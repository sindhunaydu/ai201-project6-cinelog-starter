# PR Response Doc — CineLog Watchlist Feature

## AI Usage
For Comments 4 and 5, I used Claude as a devil's advocate on my draft positions before finalizing them. I asked: "What counterargument would a careful code reviewer raise against this position? What tradeoff am I not acknowledging?"

- **Comment 4:** My draft (default `public=False`) held up, but the stress test surfaced a sharper version of the counterargument than I'd written — that apps like Letterboxd/Goodreads default "want to watch" lists to public because being browsable is the point of the feature, and defaulting private could mean the feature ships invisible. I kept my position but rewrote the tradeoff paragraph to name that comparison directly instead of a vaguer "reduced engagement" line.
- **Comment 5:** This one actually changed my answer. My first draft proposed sorting by `date_added` ascending (oldest-first) as a third option, to stop old watchlist saves from getting buried. The stress test pointed out that oldest-first buries something worse: the film you *just* added, which is the most contextually relevant item right after using the add feature — and that flipping the sort direction relative to `get_collection()` with no explanation is its own maintenance trap. I changed my recommendation to implement the maintainer's preference (`date_added` descending), with my own reasoning for why on top of the consistency argument.

I also used AI to sanity-check my final commit history before pushing: gave it the `git log --oneline origin/main..HEAD` output and asked whether the messages followed conventional commit format and whether any bundled multiple logical changes. It flagged one pre-existing commit (`fix: update film retrieval method to use db.session.get in collection and watchlist services`) as bundling the `db.session.get` fix with an incidental `app.py` import-path change. I checked it myself against the spec and agreed it's a real (if minor) violation — I left it as-is since it predates my changes and wasn't cleanly splittable with reword/squash/fixup/drop alone, but noted it in the "Commit history cleanup" section below instead of silently ignoring it.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py`, and updated the import + call in `routes/watchlist/watchlist.py`.
**Where I looked for call sites:** `grep -rn "save_to_watchlist" .` across the whole repo, not just the service file — the only other hit was the import/call in `routes/watchlist/watchlist.py`. Ran the same grep again after the rename to confirm nothing still referenced the old name.
**How I verified:** `flask run` + curled `POST /watchlist/<user_id>/add` to make sure the endpoint actually worked end to end, since a clean import doesn't prove the route still resolves the right function.

## Comment 2 — Deduplication
**What I did:** Added an `AlreadyInWatchlistError` and a `WatchlistEntry.query.filter_by(user_id=..., film_id=...)` lookup at the top of `add_to_watchlist()`, so it raises before attempting the insert instead of relying on a DB constraint to catch the duplicate.
**How I verified:** Ran the dev server and POSTed the same `user_id`/`film_id` to `/watchlist/<user_id>/add` twice — first call created the entry, second call hit `AlreadyInWatchlistError` instead of a DB integrity error. This was manual, not automated — there's no test covering it yet. Should add one modeled on `test_add_to_collection_duplicate_raises` in `tests/test_collection.py` before this merges.

## Comment 3 — Missing test
**What I did:** Added `test_add_to_watchlist_nonexistent_film_raises` in a new `tests/test_watchlist.py`.
**Test I modeled it on:** `test_add_to_collection_nonexistent_film_raises` in `tests/test_collection.py` — same shape: in-memory sqlite `app` fixture, `sample_user` fixture, `pytest.raises(FilmNotFoundError)` around the call. Copied those two fixtures into the watchlist test file since there's no shared `conftest.py` yet.
**How I verified:** `pytest tests/test_watchlist.py -v` passes, and ran the full suite (`pytest -v`) to confirm the duplicated fixtures don't clash with `test_collection.py` — all 5 tests pass.

## Comment 4 — Default visibility
**My position:** Flip `WatchlistEntry.public` to default `False` instead of `True`.

**Reasoning:** I'm optimizing for not sharing something on a user's behalf before they've had any chance to decide they want it shared. Right now `add_to_watchlist()` sets `public=True` the moment someone saves a film — before any settings screen, any explanation of what "public" means, any opt-in. That's the wrong order of operations: consent should come before exposure, not after.

It also turns out `public` isn't enforced anywhere yet — I grepped the whole codebase and the only two places it's referenced are the column definition and `to_dict()`. `GET /watchlist/<user_id>` returns every entry regardless of the flag, to anyone who hits the URL, with no auth check. So today the default has zero product benefit either way — nothing reads it as a gate. But that also means there's no cost to getting this right now, and a real cost to getting it wrong: if a future "browse your friends' watchlists" feature ships and starts filtering on `public=True`, every entry created under the current default gets retroactively exposed to that feature without the user ever making that choice. Fixing the default later means either a data migration or accepting that early users default one way and later users default the other — messier than just starting private.

**Tradeoff acknowledged:** The counterargument is real: watchlist-style "want to watch" lists are exactly the kind of content apps like Letterboxd or Goodreads default to public, because being browsable by other users is the whole social value of the feature — that's arguably why a `public` field exists on this model at all, and not on `CollectionEntry`. Defaulting to private means the feature launches invisible, and most users won't discover or flip a setting they don't know exists, so any future discovery surface built on top of it starts in an empty room. I'm accepting that cost because there's no discovery feature shipping in this PR to justify the exposure yet — when one does, it should ship its own explicit prompt ("share your watchlist with followers?") rather than inherit an opt-out that happened silently, months earlier, on a screen the user never saw.

## Comment 5 — Sort order
**My position:** Implement the maintainer's preference — sort by `date_added` descending, matching `get_collection()`.

**Reasoning:** I considered two alternatives before landing here: keeping the current alphabetical sort (a watchlist is something you scan like a menu, so title order helps you find a specific film), and flipping to `date_added` ascending — oldest saved first — to fight "watchlist rot," where old saves get buried under new ones and never watched. I ended up rejecting both. Alphabetical throws away a meaningful signal (recency) for no real gain once the list has more than a handful of films. And oldest-first solves that problem by creating a worse one: it buries the film you just added — the single most contextually relevant item, since looking at your watchlist right after adding to it is the most common interaction — under everything else on the list.

**Engagement with reviewer's point:** The maintainer's underlying point is stronger than "just be consistent for its own sake" — `WatchlistEntry` and `CollectionEntry` are structurally identical (`date_added` on both), exposed through nearly parallel `GET` endpoints, so having one sort by `Film.title` and the other by `CollectionEntry.date_added.desc()` means a future contributor can't pattern-match one against the other, which is exactly what CONTRIBUTING.md's "compare existing functions before adding new ones" convention is trying to prevent. I'm agreeing with the direction (descending, newest first) for my own reason on top of that: it's the ordering that matches what the user just did, not just what's structurally consistent.

I'm not pretending this has no cost — a film saved three months ago can sink to the bottom of a long list and effectively disappear, which is a legitimate UX gap. But the fix for that is a deliberate feature (manual reordering, a "revisit old saves" section) with its own design discussion, not a silently reversed sort direction on this endpoint. Out of scope for this PR.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin` then `git rebase origin/main`. Git stopped on the second replayed commit (`refactor(watchlist): Rename save_to_watchlist() to add_to_watchlist()`) with a real conflict marker in `.gitignore` — both branches had independently added `.venv/`/`venv/` entries, and `main` also added `.pytest_cache/`. Trivial, resolved by keeping the union of both lists.

That conflict marker wasn't the interesting part, though. Once the rebase finished cleanly (all 5 commits replayed with no further stops), the `WatchlistEntry` model was gone from `models.py` entirely. `git rebase` doesn't consider that a conflict, because none of `feature/watchlist`'s commits ever touch `models.py` after the branch's first commit — `WatchlistEntry` was present from the very first shared commit, and `main`'s UUID-migration commit (`refactor: migrate film IDs from integer to UUID`) deleted it outright as collateral cleanup while switching `Film.id` from `Integer` to `String(36)`. Since my branch had no diff against that file to conflict with, the deletion just carried through silently — a rebase can lose content without ever showing a `<<<<<<<` marker if your branch simply never touches the file that changed upstream.

**How I resolved it:** Re-added `WatchlistEntry` to `models.py`, but with `film_id = db.Column(db.String(36), db.ForeignKey("film.id"), ...)` instead of the old `db.Integer`, to match the UUID type `Film.id` now uses on `main`. Also updated the two other spots that still assumed integer film IDs: the `film_id (int)` docstring in `services/watchlist_service.py:add_to_watchlist()`, and the `Body: { "film_id": <int> }` docstring in `routes/watchlist/watchlist.py`.

While manually exercising the restored model I found a second, separate bug (not a rebase artifact — it predates this PR): `Film` only ever declared a relationship/backref for `CollectionEntry` (`collection_entries = db.relationship("CollectionEntry", backref="film", ...)`), never for `WatchlistEntry`, so `get_watchlist()`'s call to `entry.film.to_dict()` raised `AttributeError: 'WatchlistEntry' object has no attribute 'film'`. Added `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` alongside the existing one. Flagging this separately from the UUID conflict since it wasn't caused by the rebase — it just happened to be latent and only surfaced when I ran the feature end to end.

**How I verified no conflict remains:**
- `git status` shows no `rebase-merge` in progress and a clean working tree relative to the rebase (no `Unmerged paths`).
- `git log --oneline --merges main..feature/watchlist` returns nothing — no merge commits were introduced by this branch. (The one merge commit that does show up in `git log --merges` on the branch, `Merge pull request #2 from ascherj/chore/add-gitignore`, is inherited from `main`'s own history via the rebase, not created by this branch — consistent with CONTRIBUTING.md's "no merge commits" rule, which is about not merging `main` into the feature branch.)
- `pytest -v` — all 5 tests pass post-rebase.
- Manually exercised the previously-broken path in a Python shell against an in-memory DB: created a real `Film` (UUID id), called `add_to_watchlist()` with that UUID, confirmed the entry's `film_id` is stored as a `str`, called `add_to_watchlist()` again with the same ids to confirm `AlreadyInWatchlistError` still fires on a UUID `film_id`, then called `get_watchlist()` and confirmed it returns the film dict instead of raising — this is what caught the missing-relationship bug above, since none of the existing automated tests call `get_watchlist()` on a non-empty list.

## Commit history cleanup
Ran `git log --oneline origin/main..HEAD` to count commits ahead of `main` (9), then `git rebase -i HEAD~9`. Marked the three commits with problems as `reword`:
- `added watchlist model and endpoint fixed a bug more changes` → `feat: add watchlist model and add_to_watchlist endpoint` (no type prefix, past tense, vague trailing lines)
- `refactor(watchlist): Rename save_to_watchlist() to add_to_watchlist()` → `refactor(watchlist): rename save_to_watchlist() to add_to_watchlist()` (capitalized description)
- `refactor(watchlist): add deduplication logic to add_to_watchlist()` → `fix: add deduplication check to prevent duplicate watchlist entries` (adding validation is a behavior change, not a no-op refactor per CONTRIBUTING.md's own definition of `refactor:`)

Left the other six as `pick` — they were already conventional, and each represents one logical change: the `db.session.get` fix, the deduplication test, and the four fixes I made while resolving Comment 6 (UUID restoration, the missing `Film`↔`WatchlistEntry` relationship, the visibility default, and the sort order).

Before finalizing, I checked the resulting `git log --oneline` against the conventional commits spec myself: all 9 messages use a valid `type(scope): description` with types from CONTRIBUTING.md's allowed list, lowercase imperative descriptions, and (with one exception) touch only one logical concern. The exception is `fix: update film retrieval method to use db.session.get in collection and watchlist services` — a pre-existing commit (not one I authored) that bundles the `Query.get()` → `db.session.get()` modernization with an incidental `app.py` import-path fixup. It's minor and wasn't something I could cleanly split using only reword/squash/fixup/drop, so I left it and am noting it here rather than silently ignoring it.

Final history (`git log --oneline origin/main..HEAD`):
```
8591ba5 fix: sort watchlist entries by date added instead of title
9c509d1 fix: default new watchlist entries to private
e747812 fix: add missing relationship between Film and WatchlistEntry
554cad1 fix: restore WatchlistEntry model with UUID film_id after main branch refactor
e553854 test(watchlist): add unit tests for nonexistent film error
f707723 fix: add deduplication check to prevent duplicate watchlist entries
41d771e refactor(watchlist): rename save_to_watchlist() to add_to_watchlist()
58b31e0 fix: update film retrieval method to use db.session.get in collection and watchlist services
d88ab4d feat: add watchlist model and add_to_watchlist endpoint
```

<!-- SCREENSHOT: paste your own `git log --oneline` screenshot here. I ran the command above and verified the output, but I can't capture your screen, so this needs to be a real screenshot you take yourself. -->

No merge commits: `git log --oneline --merges main..feature/watchlist` returns nothing.

Pushed with `git push origin feature/watchlist --force-with-lease` after confirming `origin` was up to date via `git fetch origin` first (so the lease would actually catch a conflicting remote update if one existed).

## PR Description

**What this feature does**

Adds a watchlist to CineLog — a place for users to save films they want to watch later, separate from their collection (films already watched). Two new endpoints:

- `GET /watchlist/<user_id>` — returns a user's watchlist, most recently added film first
- `POST /watchlist/<user_id>/add` — adds a film to a user's watchlist (`{"film_id": "<uuid>"}` in the request body)

Each watchlist entry tracks `date_added` and `public` (whether the entry is visible to other users — not yet enforced anywhere, since there's no browse-other-users feature yet).

**Design decisions**

1. **Default visibility — `public` defaults to `False`.** Nothing in this codebase reads or filters on `public` yet, so defaulting it to `True` would expose user data with no offsetting product benefit today, and would silently opt users into sharing before any UI exists for them to control it. Full reasoning and the acknowledged tradeoff (a future social/discovery feature launches with nothing visible by default) is in Comment 4 above.
2. **Sort order — `date_added` descending.** `get_watchlist()` now sorts newest-first, matching `get_collection()`'s convention, rather than alphabetically by title. I considered and rejected an oldest-first alternative — full reasoning in Comment 5 above.

**How to test manually**

1. `pip install -r requirements.txt` (or use the checked-in `.venv`: `.venv/bin/pip install -r requirements.txt`)
2. Start the app: `.venv/bin/python app.py` (or `flask run`) — runs on `http://localhost:5000`
3. There's no seed data and no `POST /users` or film-creation endpoint, so create a user and a film directly in a Python shell before hitting any HTTP endpoints:
   ```
   .venv/bin/python -c "
   from app import create_app, db
   from models import User, Film
   app = create_app()
   with app.app_context():
       db.create_all()
       user = User(username='tester', email='tester@example.com')
       film = Film(title='Paddington 2', year=2017, genre='Comedy')
       db.session.add_all([user, film])
       db.session.commit()
       print('user_id:', user.id)
       print('film_id:', film.id)
   "
   ```
   Use the printed `user_id`/`film_id` in the steps below. `curl http://localhost:5000/films/` will also list it once the server is running.
4. Add a film to the watchlist:
   ```
   curl -X POST http://localhost:5000/watchlist/<user_id>/add \
     -H "Content-Type: application/json" \
     -d '{"film_id": "<film-uuid>"}'
   ```
   Expect `201` with the new entry, and `"public": false` in the response.
5. View the watchlist: `curl http://localhost:5000/watchlist/<user_id>` — should list the film just added.
6. Add a second film, then re-`GET` the watchlist — the second film should appear first (newest-added-first sort).
7. Repeat step 4 with the same `user_id`/`film_id` pair — `add_to_watchlist()` raises `AlreadyInWatchlistError` for this; note the route doesn't currently catch it into a clean 4xx response (unlike `routes/collection.py`), so you'll see a 500 with a traceback rather than a JSON error body — this is a known pre-existing gap, not something this PR claims to fix.
8. Repeat step 4 with a made-up `film_id` — same caveat: `FilmNotFoundError` is raised but not caught in the route, so expect a 500, not a clean 404.
9. Run the automated suite: `.venv/bin/python -m pytest -v` — all 5 tests should pass.
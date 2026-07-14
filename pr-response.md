# PR Response Doc — CineLog Watchlist Feature

## AI Usage
<!-- Fill in at the end — how you used AI tools during this project -->
1. **Stress-testing Comments 4 and 5 (design decisions).** After writing my own first-draft position and reasoning for each, I gave the draft to an AI acting as a skeptical reviewer and asked: "What counterargument would a careful code reviewer raise against this position? What tradeoff am I not acknowledging?" For Comment 4, it flagged that (a) treating collection's missing privacy field as precedent risks justifying an oversight rather than fixing it, and (b) an opt-in override doesn't really mitigate a risky default since most users never touch it — both are now addressed head-on in my response rather than glossed over. For Comment 5, it flagged a tie-breaking gap when two entries share a `date_added` timestamp, which I fixed by adding `id` as a secondary sort key, and noted that deferring alphabetical lookup to a future `sort` param is a deferral, not a resolved tradeoff, which I now say explicitly instead of implying it's handled. I did not ask AI to write either argument — both drafts were mine first; AI was only used to find holes in reasoning I'd already committed to.
2. **Verifying conventional commit format (Comment 6 / history rewrite).** Before finalizing the rewritten history, I gave the final `git log --oneline` output to an AI and asked whether the messages followed Conventional Commits format and whether any bundled multiple logical changes. It raised three points: (a) the rename commit (`fix: rename save_to_watchlist...`) is arguably `refactor:` rather than `fix:` since it's a pure rename with no behavior change; (b) the dedup commit is arguably `feat:` rather than `fix:` since it adds new validation behavior; (c) the final UUID-restoration commit bundles "restoring a deleted model" and "using the UUID type" as two logical changes. I checked all three against the actual Conventional Commits spec myself before acting on them, and kept my original wording for (a) and (b) — both match the assignment's own sanctioned example commit list verbatim, and I judge "fix" defensible in both cases (rename was reviewer-mandated as a defect relative to project convention; dedup corrects a gap relative to the established `add_to_collection()` invariant). I rejected (c) on inspection: there's no valid intermediate state where the model is restored with the old `Integer` type and separately migrated to `String(36)` — the foreign key has to match `Film.id`'s type the moment the model exists again, so splitting it would require committing a broken intermediate state on purpose. I did not accept any AI suggestion without checking it against the spec and the actual diffs first.
3. **Codebase orientation.** I read `services/collection_service.py`, `models.py`, and `tests/test_collection.py` directly (not via AI) before touching any review comment, per the project's own hint that understanding existing patterns first prevents misreading what a comment is asking for. I did not use AI to summarize the codebase for me.

## Comment 1 — Rename
**What I did:** Renamed `save_to_watchlist()` to `add_to_watchlist()` in `services/watchlist_service.py` to match the project's `verb_to_noun` convention used by `add_to_collection()`. Updated the single call site in `routes/watchlist/watchlist.py` (both the import and the function call).
**How I verified:** Ran `grep -rn "save_to_watchlist" --include="*.py" .` (excluding `.venv`) before and after the change. Before: three matches (the definition and one import + one call in the route file). After: zero matches. Also ran `pytest tests/ -v` to confirm the route still imports and functions correctly.

## Comment 2 — Deduplication
**What I did:** Mirrored the pattern in `add_to_collection()`: before creating a `WatchlistEntry`, query for an existing entry with the same `user_id`/`film_id`. If one exists, raise a new `AlreadyInWatchlistError` (parallel to `AlreadyInCollectionError`) instead of silently inserting a duplicate. Also updated `routes/watchlist/watchlist.py` to catch this exception and return `409 Conflict`, and to catch `FilmNotFoundError` and return `404` — the route previously didn't handle either exception and would have surfaced an unhandled 500, unlike the collection route it's supposed to mirror.
**How I verified:** Ran an ad-hoc script that adds the same film to a user's watchlist twice and confirmed the second call raises `AlreadyInWatchlistError` rather than creating a duplicate row. Also ran the full `pytest tests/ -v` suite to confirm no regressions in the collection tests.

## Comment 3 — Missing test
**What I did:** Created `tests/test_watchlist.py`, modeled directly on `tests/test_collection.py`. It reuses the same `app`/`sample_user`/`sample_film` fixture structure. The specifically requested test, `test_add_to_watchlist_nonexistent_film_raises`, is the direct equivalent of `test_add_to_collection_nonexistent_film_raises`: it calls `add_to_watchlist()` with a fake film id and asserts `FilmNotFoundError` is raised. I also added `test_add_to_watchlist_creates_entry` and `test_add_to_watchlist_duplicate_raises` (mirroring the collection suite's basic-add and dedup tests) so the new service has the same baseline coverage `collection_service.py` has, not just the one test that was explicitly requested.
**How I verified:** Ran `pytest tests/test_watchlist.py -v` (all 3 pass) and then the full suite `pytest tests/ -v` (7/7 pass, no regressions to the collection tests).

## Comment 4 — Default visibility
**My position:** Keep the default as `public=True`.

**Reasoning:** I looked at how visibility is handled elsewhere in the schema before deciding. `CollectionEntry` (the "films I've already watched" list) has no `public` field at all — there is no visibility control on it whatsoever, which means a user's watch history is *implicitly* fully public today, with no way to hide it. Given that precedent, defaulting the watchlist to `public=True` is actually the more consistent choice, not an arbitrary inherited default: it keeps the two list types behaviorally aligned (both visible by default) rather than introducing a watchlist that is quietly private while the collection next to it has no privacy concept at all. CineLog is also positioned as a *community* film-tracking app — the value of a watchlist in that kind of product (à la Letterboxd) comes largely from other users being able to see what you're planning to watch, which drives the social/discovery loop the app is built around. A private-by-default watchlist would undercut that without anyone having made a deliberate call to prioritize privacy over discovery.

**Tradeoff acknowledged:** The real cost is that a watchlist can reveal more than a collection does, in a specific way: it exposes *intent*, not just history. Films someone has already watched are, in a sense, already "claimed" — but a want-to-watch list can surface sensitive or private interests (e.g., a documentary about a health condition, or something the user hasn't decided if they want associated with their public profile) before the user has actually engaged with it. Defaulting that to public risks surprising users who reasonably assume "I haven't watched this yet" also means "no one can see I'm interested in it."

I stress-tested this with AI acting as a skeptical reviewer, and it raised two points I want to engage with directly rather than gloss over:

1. *"Collection has no privacy field" might be an oversight, not a deliberate baseline — you're using a gap to justify propagating the gap."* This is fair, and I don't think consistency alone settles the question. I'm not claiming collection's lack of a privacy control is correct — only that it tells us no one has yet decided watch-history should be private in this app, so treating watchlist as the place to unilaterally introduce a stricter, opt-out-by-default privacy model (without also revisiting collection) would be an inconsistent, half-applied policy, not a considered one. If CineLog wants privacy-by-default, that's a product decision that should apply to both entry types together — it shouldn't be smuggled into just the field this PR happens to be touching.

2. *"An opt-in `public` override only protects users who know to use it — defaults dominate real behavior, so the override doesn't actually mitigate the risk, it just makes it theoretically escapable."* This is the strongest objection and I can't fully rebut it: a frontend that doesn't prominently surface the toggle will produce the same outcome as if the override didn't exist. I'm not going to claim the override "solves" the privacy risk — it doesn't. What it does is move the decision from "made invisibly by a default the user never sees" to "made explicitly in the API surface, visible to whoever builds the client." That's a real but partial mitigation, not a resolution, and I'm accepting that residual risk in exchange for keeping the community/discovery behavior the reviewer and I both think this feature needs by default.

**Implementation:** Added an optional `public` parameter to `add_to_watchlist(user_id, film_id, public=True)` and wired it through the `POST /watchlist/<user_id>/add` endpoint body (`{"film_id": ..., "public": false}`), so a caller can opt out of the default per-entry instead of it being silently fixed. Covered by `test_add_to_watchlist_creates_entry` (asserts the default is `True`) and `test_add_to_watchlist_respects_explicit_public_override` (asserts `public=False` persists) in `tests/test_watchlist.py`. `pytest tests/ -v` — 9/9 passing.

## Comment 5 — Sort order
**My position:** I agree with the reviewer. I changed `get_watchlist()` to sort by `date_added` descending (newest first) instead of alphabetical by title.

**Reasoning:** Beyond the reviewer's UX argument, there's a project-consistency argument that pushed me firmly to this choice: `get_collection()` already sorts by `CollectionEntry.date_added.desc()` — it's not just convention, it's directly tested (`test_get_collection_returns_newest_first` in `test_collection.py`). Having the collection view be "newest first" and the watchlist view be "alphabetical" would mean the two most closely related endpoints in the app behave differently for no functional reason, which is a worse outcome than either choice applied consistently. Since the reviewer's UX instinct and the existing project convention point the same direction, alphabetical was the harder position to defend, not the reviewer's preference.

**Engagement with reviewer's point:** The reviewer's core claim is "most users want to see what they added recently," which I think is specifically true for a *watchlist* in a way it might not be for, say, a film database sorted by title — a watchlist's primary use case is "what should I watch next," and the most recently added film is the one most likely to be top-of-mind, since it's what the user was just thinking about when they added it. Alphabetical order optimizes for lookup ("is X on my list?") over decision-making ("what do I watch tonight?"), and lookup is a secondary use case here. The one scenario alphabetical order genuinely helps — a user with a large watchlist trying to find one specific title — is better solved by a search/filter feature or an explicit `?sort=` query param later, not by making the default sort order optimize for the rarer case.

I stress-tested this draft with AI as a skeptical reviewer and it raised a gap I hadn't considered: sorting by `date_added` alone is nondeterministic when two entries share the same timestamp (plausible for watchlists specifically, since a user might add several films in the same request/second in a way they wouldn't for one-at-a-time collection logging). I fixed this by adding `WatchlistEntry.id` as a secondary sort key (`.order_by(WatchlistEntry.date_added.desc(), WatchlistEntry.id.desc())`) so ties resolve deterministically. It also pointed out that "defer lookup to a future `?sort=` param" is a deferral, not a resolved tradeoff — that's fair; I'm not committing to build that param in this PR, and if it never gets built, alphabetical browsing simply isn't supported. I think that's an acceptable gap for a v1 (the reviewer can weigh in if they disagree), but I'm flagging it here rather than implying it's handled. I added `test_get_watchlist_returns_newest_first` to `tests/test_watchlist.py` (mirroring `test_get_collection_returns_newest_first`) to lock in the decision the same way the collection behavior is locked in.

**Bug found while writing this test:** Writing `test_get_watchlist_returns_newest_first` was the first thing in this whole PR to actually call `get_watchlist()` and inspect its output (the earlier tests only exercise `add_to_watchlist()`). It failed immediately with `AttributeError: 'WatchlistEntry' object has no attribute 'film'` — `Film` declares `collection_entries = db.relationship("CollectionEntry", backref="film", ...)` but has no equivalent relationship for `WatchlistEntry`, so `entry.film.to_dict()` in `get_watchlist()` could never have worked. I added `watchlist_entries = db.relationship("WatchlistEntry", backref="film", lazy=True)` to the `Film` model, matching the existing `collection_entries` pattern exactly. This was a pre-existing bug, not something introduced by the sort-order change, but it was only surfaced by adding real coverage of `get_watchlist()`.

## Comment 6 — Rebase
**What conflicted:** Ran `git fetch origin && git rebase origin/main`. There were two distinct problems, only one of which git actually flagged:

1. **Flagged conflict — `.gitignore`:** both `main` (via a separate merged PR, `chore: add .gitignore for generated files`) and my branch's first replayed commit added a `.gitignore` with slightly different content (mine additionally ignored `.pytest_cache/`). Git reported this as an add/add conflict.
2. **Silent, unflagged conflict — `models.py`:** `main`'s `refactor: migrate film IDs from integer to UUID` commit changed `Film.id` from `Integer` to `String(36)` *and*, in the same commit, deleted the `WatchlistEntry` model entirely (it existed in the pre-refactor state but wasn't part of `main`'s feature set, so the refactor author dropped it rather than migrating it). None of my branch's commits touch that hunk of `models.py` directly — they only import and use `WatchlistEntry` from `services/watchlist_service.py` — so git replayed the deletion cleanly with **no conflict marker at all**. `git rebase` finished with "Successfully rebased," but `WatchlistEntry` was gone from `models.py`, and `pytest` immediately failed on `ImportError: cannot import name 'WatchlistEntry' from 'models'`.

**How I resolved it:** For the `.gitignore` conflict, I kept the union of both versions (`.pytest_cache/`, `.venv/`, `venv/`) since both entries are legitimate. For the silently-dropped model, I re-added the `WatchlistEntry` class to `models.py` after the rebase completed, using `db.Column(db.String(36), db.ForeignKey("film.id"), nullable=False)` for `film_id` — matching exactly how `main`'s refactor updated `CollectionEntry.film_id` to the new UUID type — instead of the old `db.Integer`. I committed this as its own follow-up commit rather than trying to fold it into the rebase machinery, since it's a real code change (restoring a deleted model), not a conflict-resolution artifact.

**How I verified no conflict remains:** `git log main..feature/watchlist --merges --oneline` returns nothing, confirming no merge commits were introduced. `pytest tests/ -v` passes all 9 tests post-rebase, including the two that would fail loudest if the UUID migration were incomplete (`test_add_to_watchlist_nonexistent_film_raises`, which uses a UUID-formatted fake id, and `test_get_watchlist_returns_newest_first`, which round-trips real `Film` rows through `WatchlistEntry`). I also ran `grep -rn "int\b\|Integer" services/watchlist_service.py routes/watchlist/watchlist.py tests/test_watchlist.py models.py` and confirmed the only remaining `Integer` columns (`Film.year`, `CollectionEntry.rating`) are legitimately integers, unrelated to film IDs.

## Final Commit History
<!-- `git log --oneline main..feature/watchlist` — no screenshot tool was available in this environment, so the raw terminal output is included below as the equivalent evidence. -->

```
$ git log --oneline main..feature/watchlist
d49f3e9 docs: add pr-response.md documenting review responses and design decisions
03ab67d fix: restore WatchlistEntry model with UUID film_id after main rebase
5205f79 feat: add public parameter to add_to_watchlist for explicit visibility control
5a3ba4e fix: sort get_watchlist by date_added descending to match get_collection()
03f8ef7 fix: add missing Film.watchlist_entries relationship
a3957e9 test: add watchlist test coverage for creation, dedup, and missing film
924c092 fix: add deduplication check to prevent duplicate watchlist entries
ba89e4d fix: rename save_to_watchlist to add_to_watchlist per naming convention
96361ab fix: update film retrieval method to use db.session.get in collection and watchlist services
1f09f22 feat: add watchlist model, service, and endpoint
```

`git log main..feature/watchlist --merges --oneline` returns nothing — no merge commits on the branch.

## PR Description
<!-- Written at the end — feature overview, design decisions, manual testing steps -->

### What this adds

This PR adds a **watchlist** feature: users can save films they want to watch later, view their watchlist, and each entry can be marked public or private. It follows the same model/service/route structure as the existing `collection` feature (films already watched).

- `WatchlistEntry` model (`user_id`, `film_id`, `date_added`, `public`)
- `add_to_watchlist(user_id, film_id, public=True)` — creates an entry, rejects nonexistent films (`FilmNotFoundError` → 404) and duplicates (`AlreadyInWatchlistError` → 409)
- `get_watchlist(user_id)` — returns a user's watchlist sorted newest-added-first
- `POST /watchlist/<user_id>/add` and `GET /watchlist/<user_id>` endpoints

### Design decisions (see `pr-response.md` for full reasoning)

- **Default visibility (Comment 4):** New watchlist entries default to `public=True`. This keeps the watchlist consistent with the fact that `CollectionEntry` (already-watched films) has no privacy control at all today, and matches CineLog's identity as a community app where watchlist visibility drives discovery. Callers can override this per-entry via the new `public` parameter on `add_to_watchlist()` / the add endpoint (`{"film_id": ..., "public": false}`).
- **Sort order (Comment 5):** `get_watchlist()` sorts by `date_added` descending (newest first, with `id` as a tie-breaker), matching `get_collection()`'s existing tested convention and optimizing for the watchlist's primary use case ("what should I watch next") over alphabetical lookup.

### Manual testing steps

1. `pip install -r requirements.txt` (or activate the project's venv) and `pytest tests/ -v` — all 9 tests should pass.
2. Start the app: `python app.py`
3. Create a user and a film (via existing endpoints, or directly via a Python shell using `create_app()`/`db.session`), noting their UUIDs.
4. `POST /watchlist/<user_id>/add` with body `{"film_id": "<film-uuid>"}` — expect `201` and a JSON entry with `"public": true`.
5. Repeat the same request — expect `409` with an "already on this user's watchlist" error, not a duplicate row.
6. `POST /watchlist/<user_id>/add` with a nonexistent `film_id` — expect `404`.
7. `POST /watchlist/<user_id>/add` with `{"film_id": "<other-film-uuid>", "public": false}` — expect `201` with `"public": false`.
8. `GET /watchlist/<user_id>` — expect both films, most recently added first.

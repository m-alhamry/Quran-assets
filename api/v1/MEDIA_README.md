# Quran image and audio catalogue

The app supports the existing two page layouts and 38 per-ayah reciters through
`Quran-assets/api/v1/media-catalog.json`, served by GitHub Raw with jsDelivr as
fallback. This is metadata for the existing media repositories, not a new content
licence. Publishing or adding media still requires the applicable source rights.

## App behaviour

- The bundled seed and last validated remote catalogue work offline. Metadata
  checks happen when media is requested, at most every 12 hours in a process,
  with successful checks persisted across restarts. Failed network checks never
  block reading existing files. No startup dependency is added.
- A versioned gzip manifest for each edition lists exact file sizes and SHA-256;
  image records also include PNG dimensions. Manifests are fetched on demand and
  cached; they are not all bundled or loaded into memory. Decoding occurs off the
  UI isolate. Retired manifest cache versions are pruned on catalogue refresh.
- Media URLs pin a Git commit. Both managed mirrors must contain identical bytes.
  Revisions cannot change the supported edition IDs, page counts, layout contract,
  ayah-per-file format or basmala contract. Changed layouts/recordings require a
  new reader integration, not just editing a compatibility label.
- Existing URL-based image cache keys, reciter IDs, audio DB/loose-file paths,
  selected reciters, last-read positions and Khatmah editions are retained. Existing
  downloads are read as before, without a compulsory checksum scan or redownload.
  New source metadata affects missing/new downloads. To replace an already cached
  edition deliberately, use its existing delete/download controls.
- Audio download batches and playback sessions hold a fixed catalogue snapshot.
  Continuous-page enqueue uses that same snapshot. Online ayahs use a
  `StreamAudioSource`: fetch and validate a complete small ayah, fall back before
  exposing bytes, then serve native byte ranges. This avoids stitching different
  recordings together during a seek. At most three recent audio futures are cached.
  Native Play/Pause/Stop, metadata, repeat and recovery logic remain unchanged.
- Source switches don't change the selected ayah or position. If every mirror
  fails, existing playback recovery exposes Retry. Pause/Stop are never reversed
  by transport completion. Offline audio continues to use its current DB/files.
- Native whole-surah/timing-based playback code remains available for legacy
  formats but is not remotely configurable in this protocol.
- The old third-party routes (EveryAyah/android.quran.com) remain in the legacy
  URL helpers, but are not used by managed production transfers. Their differently
  encoded files/layouts are not interchangeable checksum mirrors. Add them only
  with separately verified compatible manifests; never relax verification to
  accept an unrelated recording/page. Tier-3-only developer mode therefore has no
  managed source. Normal production tier mode remains `all`.

## Publishing a metadata update

1. Review the source rights, exact recording/layout and completeness. Commit media
   changes in the relevant existing repository first. Do not relabel incompatible
   media as the same layout/recording.
2. From the app root, build with a strictly higher revision:

   ```sh
   python3 scripts/quran/build_media_catalogue.py --revision 2
   ```

   This reads `../app_repo/Quran-assets` and `Quran-audio-assets-1` through `-6`.
   It produces `build/media-catalogue/api/v1/` and refreshes the app's small seed
   at `assets/data/quran/media_catalogue.json`. It never copies audio or images.
   It rejects uncommitted media edits, uses committed revisions in media URLs,
   and records missing files explicitly instead of inventing content.
3. Review the diff. Copy the generated `api/v1/media/` manifests and
   `api/v1/media-catalog.json` into the existing `Quran-assets` checkout. Preserve
   all previously published immutable manifests and media commits. Commit/push
   these metadata files together, leaving unrelated repository files untouched.
4. Check both public catalogue endpoints, every manifest hash, and representative
   image/audio downloads. Rollback uses a higher catalogue revision pointing at
   the previous manifests/commits. Reordering or repairing compatible sources
   requires no app release after this integration ships. New protocols do.

## Known existing source holes

Al-Ajami (`113`) lacks `033050.mp3` and `074040.mp3` in the current GitHub media
copy. The matching EveryAyah responses retrieved on 2026-10-03 contain ID3 tags
but no decodable MP3 frames (ffprobe also rejects them). The manifest explicitly
marks these two files unavailable. Existing user-downloaded files are retained;
new requests for these verses fail instead of caching broken audio or silently
substituting another reciter. Repair requires complete files of the same recording.

## Checks

```sh
flutter test test/features/quran
QURAN_MEDIA_LOCAL_ROOT=../app_repo flutter test test/features/quran/media_catalogue_local_test.dart
flutter analyze
```

Ready for review
Select text to add comments on the plan
Live-shared analysis inputs (Firestore sync)
Context
index.html is a single static file with no backend — the only persisted state today is the dark/light theme, saved per-browser to localStorage (initThemeToggle(), lines 1543-1578). Everything else — sliders, numeric fields, holiday-mode/financing radios, ramp/HVAC checkboxes — lives only in the DOM and resets per browser/session. readState() (812-834) rebuilds a snapshot from the DOM on every recalc() (1353-1504), which is the single entry point for both computation and DOM output.

The user wants those analysis inputs to be live-shared: a change in one browser should show up for anyone else currently viewing the page, with no login/auth required to write (their explicit choice), but without an XSS/injection hole or wide-open malformed writes. That means the security has to come from server-side validation of what can be written, not from restricting who can write.

Approach
Firestore (Firebase, free Spark tier) — a single document holds the shared state, onSnapshot pushes live updates to every open tab, and Firestore Security Rules validate the exact shape/type/range of every write server-side, without any auth. This fits a zero-build static file (CDN ESM import, no bundler) and needs no server code of our own.

Repo already has a GitHub remote (github.com/johnkamm/Fulton-Analysis), so GitHub Pages is the easiest hosting path to get this off file:// (Firestore needs https:// to be reliable) — no new console/product beyond Firebase itself.

Manual setup (user must do outside of code — cannot be provisioned by an agent)
Create a free Firebase project (Spark plan, no billing account).
Enable Cloud Firestore (Native mode), any region.
Register a Web App in the project to get the firebaseConfig object (safe to commit — it's not a secret; Rules are the actual gate).
Paste the security rules below into Firestore → Rules → Publish — a fresh project defaults to either fully-locked or a temporary fully-open "test mode," so this step is not optional.
GitHub repo → Settings → Pages → Deploy from branch (main/root) to get an https:// URL.
Optional later hardening: Firebase App Check (invisible reCAPTCHA v3) to reduce bot/script abuse — no login friction added.
Security rules
Single fixed doc /sharedState/analysisInputs. Public read, exact-shape validated write, no delete, deny everything else including list (blocks collection enumeration):

rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /sharedState/analysisInputs {
      allow get: if true;
      allow list: if false;
      allow create, update: if isValidState(request.resource.data);
      allow delete: if false;
    }
    match /{document=**} { allow read, write: if false; }
  }
}
isValidState() requires hasOnly + hasAll against the complete known field list (every id in sliderIds, fieldIds, the 3 radio group names, and the 2 checkbox ids), with a type + range/enum check per field — numeric ranges seeded from each input's existing HTML min/max where present, sane ceilings elsewhere (e.g. percentages 0–100, dollar fields 0–50,000,000, term-year fields 0–60). holidayMode in ["capitalize","interestOnly","simple"], permFinancing in ["mortgage","tier2","bank"], portFinancing in ["private","tier2"]. Since the synced state is 100% numbers/enums/booleans (no free-text fields), Firestore's own type/range/enum checks fully close the injection surface — the client also only ever writes remote values via .value = / .checked =, never innerHTML. Firestore range comparisons are false for NaN, so no separate NaN guard is needed.

Optional: stamp updatedAt: request.time on writes for basic auditability without auth.

Commit a copy as firestore.rules in the repo root so the rules are version-controlled even though deployment is a console paste.

Code changes — index.html
Small refactor for reuse (no behavior change): extract the inline radio-group names and checkbox ids currently hardcoded at lines 1511 and 1513-1516 into arrays next to sliderIds/fieldIds (~line 747):

var radioGroupNames = ["holidayMode", "permFinancing", "portFinancing"];
var checkboxIds = ["rampEnabled", "hvacCapexOverrideEnabled"];
Rewrite 1511-1516 to iterate these arrays instead of the literals, so one source of truth feeds both the existing listeners and the new sync code.

Expose a bridge object just before recalc(); (line 1580), since the new sync code lives in a separate type="module" script and can't otherwise see the existing IIFE's locals:

window.__sensitivitySync = {
  sliderIds: sliderIds, fieldIds: fieldIds,
  radioGroupNames: radioGroupNames, checkboxIds: checkboxIds,
  recalc: recalc
};
New <script type="module"> block after the existing </script> (line 1582), before </body>:

Import initializeApp, getFirestore, doc, onSnapshot, setDoc, serverTimestamp from the gstatic.com/firebasejs/<version>/firebase-{app,firestore}.js CDN ESM URLs; paste firebaseConfig; docRef = doc(db, "sharedState", "analysisInputs").
readRawDomState() — reads raw .value/.checked for every id/group in the bridge object (sync exactly what's in the DOM, not the ÷100-converted s values from readState(), so no re-derivation is needed on the way back in).
applyRemoteState(data) — sets .value/.checked from data for each field, then calls sync.recalc() once. Programmatic .value/.checked assignment doesn't fire input/change, so this can't loop back into the write-listeners below.
debounce(fn, 400) wrapping pushLocalState; attach as a second input/change listener on the same elements already wired at 1506-1516 — local UI stays instant (existing listener untouched), only the network write is debounced. pushLocalState() does setDoc(docRef, { ...readRawDomState(), updatedAt: serverTimestamp() }) (full overwrite, matching the rules' exact-shape check), wrapped in .catch() so a rejected/offline write never throws.
onSnapshot(docRef, { includeMetadataChanges: true }, snap => { if (snap.metadata.hasPendingWrites || !snap.exists()) return; applyRemoteState(snap.data()); }) — hasPendingWrites filters out the local-cache echo of our own optimistic write, so only server-confirmed state (ours or someone else's) gets applied.
First load: if !snap.exists(), setDoc the current DOM defaults once (harmless if two tabs race — both write the same defaults).
Verification
Live sync, two browser windows/profiles side by side:

Drag a slider / edit a number / flip a radio / toggle each checkbox in A → confirm B's control and all derived outputs (including the Section 5 holidayMonths2 mirror) update within ~1s with no refresh.
Type rapidly in a field in A → Firestore console Data tab shows one/a few writes, not one per keystroke (debounce working).
Reload B mid-session → loads the last-synced value, not the HTML default (initial-load path works, not just live-update path).
DevTools → Network → Offline in A, edit a value → local recalc() still updates instantly; go back online → write flushes, B receives it.
Rules actually reject bad writes (the real check, since it bypasses the page's JS entirely):

Firebase console Rules Playground: simulate an update with an out-of-range number, an unrecognized enum string, and an extra unknown key → expect Deny each time; simulate a valid full-shape payload → expect Allow.
REST smoke test with no auth token against https://firestore.googleapis.com/v1/projects/<PROJECT_ID>/databases/(default)/documents/sharedState/analysisInputs: malformed body → expect HTTP 403 PERMISSION_DENIED; valid body → expect success.
Attempt a runQuery/list against the sharedState collection → confirm denial (allow list: if false).
Files touched
index.html — refactor (id-list extraction), bridge object, new module script block.
firestore.rules (new) — version-controlled copy of the published rules.
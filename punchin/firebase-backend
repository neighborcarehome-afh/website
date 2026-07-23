// firebase-backend.js
// Exposes window.FB with everything index.html calls: login, logout, onAuthChange,
// getProfile, listCompanyAccounts, createAccount, sendReset, setAccountRole,
// removeAccount, loadEntries, saveEntries, loadGrid, saveGrid.
//
// >>> FILL THIS IN with your actual config from Firebase Console
//     (Project settings -> General -> Your apps -> SDK setup and configuration) <<<
const firebaseConfig = {
  apiKey: "AIzaSyAgceNDUpzdP58GDUoVVRbo5klNmRNoGWw", // seen in your network logs, already public info
  authDomain: "punch-b5e7c.firebaseapp.com",
  projectId: "punch-b5e7c",
  storageBucket: "punch-b5e7c.firebasestorage.app",   // double-check against the SDK snippet (see note below)
  messagingSenderId: "429105848962",
  appId: "1:429105848962:web:85649040bbe33159d04a9b"
};

import { initializeApp, deleteApp } from "https://www.gstatic.com/firebasejs/10.13.0/firebase-app.js";
import {
  getAuth,
  setPersistence,
  browserSessionPersistence,
  signInWithEmailAndPassword,
  signOut,
  onAuthStateChanged,
  sendPasswordResetEmail,
  createUserWithEmailAndPassword
} from "https://www.gstatic.com/firebasejs/10.13.0/firebase-auth.js";
import {
  getFirestore,
  doc,
  getDoc,
  setDoc,
  updateDoc,
  deleteDoc,
  addDoc,
  collection,
  query,
  where,
  getDocs
} from "https://www.gstatic.com/firebasejs/10.13.0/firebase-firestore.js";

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

// Session-only persistence: closing the tab/browser logs the user out.
// (Matches the comment in index.html: "Auth persistence is session-only".)
const persistenceReady = setPersistence(auth, browserSessionPersistence);

async function profileOf(uid) {
  const snap = await getDoc(doc(db, "profiles", uid));
  return snap.exists() ? { uid, ...snap.data() } : null;
}

const FB = {
  // ---------- Auth ----------
  async login(email, password) {
    await persistenceReady;
    const cred = await signInWithEmailAndPassword(auth, email, password);
    // Force-refresh the ID token before touching Firestore. Right after
    // sign-in there can be a brief race where Firestore's internal auth
    // listener hasn't picked up the new token yet, causing a spurious
    // "Missing or insufficient permissions" on this very first read even
    // though isSelf(uid) should trivially allow it.
    await cred.user.getIdToken(true);
    const profile = await profileOf(cred.user.uid);
    if (!profile) {
      // Auth account exists but no Firestore profile doc — treat as invalid.
      await signOut(auth);
      throw new Error("No profile found for this account.");
    }
    return profile;
  },

  async logout() {
    await signOut(auth);
  },

  onAuthChange(callback) {
    onAuthStateChanged(auth, callback);
  },

  async getProfile(uid) {
    return profileOf(uid);
  },

  async sendReset(email) {
    await sendPasswordResetEmail(auth, email);
  },

  // ---------- Company roster ----------
  async listCompanyAccounts(companyId) {
    const q = query(collection(db, "profiles"), where("companyId", "==", companyId));
    const snap = await getDocs(q);
    const out = {};
    snap.forEach(d => { out[d.id] = d.data(); });
    return out;
  },

  // Creates a new Auth user + their profile doc WITHOUT logging the admin out.
  // Uses a secondary, throwaway Firebase App instance for the Auth call so the
  // *primary* app's auth state (the admin's session) never changes. The
  // Firestore write below therefore still runs as the admin, which is required
  // by the security rule: allow create: if isMaster() || isAdminOf(companyId).
  async createAccount({ email, password, name, role, companyId }) {
    const secondaryApp = initializeApp(firebaseConfig, "Secondary-" + Date.now());
    const secondaryAuth = getAuth(secondaryApp);
    try {
      const cred = await createUserWithEmailAndPassword(secondaryAuth, email, password);
      const uid = cred.user.uid;
      try {
        // Written using the PRIMARY app -> request.auth is still the admin.
        await setDoc(doc(db, "profiles", uid), { email, name, role, companyId });
      } catch (firestoreErr) {
        // Roll back the orphaned auth user so we don't leave a login-but-no-profile account.
        try { await cred.user.delete(); } catch (_) { /* best effort */ }
        throw firestoreErr;
      }
      return { uid };
    } finally {
      // Sign out of and tear down the secondary app; never touches the admin's session.
      try { await signOut(secondaryAuth); } catch (_) {}
      await deleteApp(secondaryApp);
    }
  },

  async setAccountRole(uid, newRole) {
    await updateDoc(doc(db, "profiles", uid), { role: newRole });
  },

  // NOTE: this removes the Firestore profile (so they lose access to the app),
  // but it does NOT delete their Firebase Auth account — client-side code can't
  // delete another user's Auth account without the Admin SDK. To fully delete
  // the Auth account too, add a Cloud Function using firebase-admin's
  // admin.auth().deleteUser(uid) and call it here instead.
  async removeAccount(uid) {
    await deleteDoc(doc(db, "profiles", uid));
  },

  // ---------- Punch entries ----------
  async loadEntries(uid) {
    const snap = await getDoc(doc(db, "entries", uid));
    return snap.exists() ? (snap.data().entries || []) : [];
  },

  async saveEntries(uid, companyId, entries) {
    await setDoc(doc(db, "entries", uid), { companyId, entries }, { merge: true });
  },

  // ---------- Schedule grid ----------
  async loadGrid(companyId) {
    const snap = await getDoc(doc(db, "schedules", companyId));
    return snap.exists() ? (snap.data().grid || {}) : {};
  },

  async saveGrid(companyId, grid) {
    await setDoc(doc(db, "schedules", companyId), { grid }, { merge: true });
  },

  // ---------- Punch edit requests (user proposes, admin approves) ----------
  // type: 'add' | 'edit' | 'delete'
  // For 'edit'/'delete', pass idx + originalClockIn/originalClockOut (the
  // punch's current values at request time) so approveEditRequest can detect
  // if the punch changed under someone else before this gets reviewed.
  async createEditRequest({ uid, companyId, type, idx = null, proposedClockIn = null, proposedClockOut = null, originalClockIn = null, originalClockOut = null }) {
    const ref = await addDoc(collection(db, "editRequests"), {
      uid, companyId, type, idx,
      proposedClockIn, proposedClockOut,
      originalClockIn, originalClockOut,
      status: "pending",
      createdAt: new Date().toISOString(),
      reviewedAt: null,
      reviewedBy: null
    });
    return ref.id;
  },

  // Sorted newest-first client-side (avoids needing a Firestore composite index).
  async listEditRequestsForCompany(companyId) {
    const q = query(collection(db, "editRequests"), where("companyId", "==", companyId));
    const snap = await getDocs(q);
    const out = [];
    snap.forEach(d => out.push({ id: d.id, ...d.data() }));
    out.sort((a, b) => (b.createdAt || "").localeCompare(a.createdAt || ""));
    return out;
  },

  async listEditRequestsForUser(uid) {
    const q = query(collection(db, "editRequests"), where("uid", "==", uid));
    const snap = await getDocs(q);
    const out = [];
    snap.forEach(d => out.push({ id: d.id, ...d.data() }));
    out.sort((a, b) => (b.createdAt || "").localeCompare(a.createdAt || ""));
    return out;
  },

  // Applies the requested change to the employee's entries doc, then marks
  // the request approved. Throws Error("stale-request") if the target punch
  // no longer matches what it was when the request was filed (e.g. an admin
  // or a later approval already changed it) — caller should surface that
  // instead of silently applying to the wrong entry.
  async approveEditRequest(request, reviewerUid) {
    const entries = await FB.loadEntries(request.uid);
    if (request.type === "edit" || request.type === "delete") {
      const current = entries[request.idx];
      const matches = current
        && current.clockIn === request.originalClockIn
        && (current.clockOut || null) === (request.originalClockOut || null);
      if (!matches) throw new Error("stale-request");
    }
    if (request.type === "add") {
      entries.push({ clockIn: request.proposedClockIn, clockOut: request.proposedClockOut || null });
    } else if (request.type === "edit") {
      entries[request.idx] = { clockIn: request.proposedClockIn, clockOut: request.proposedClockOut || null };
    } else if (request.type === "delete") {
      entries.splice(request.idx, 1);
    }
    await FB.saveEntries(request.uid, request.companyId, entries);
    await updateDoc(doc(db, "editRequests", request.id), {
      status: "approved",
      reviewedAt: new Date().toISOString(),
      reviewedBy: reviewerUid
    });
  },

  async rejectEditRequest(requestId, reviewerUid) {
    await updateDoc(doc(db, "editRequests", requestId), {
      status: "rejected",
      reviewedAt: new Date().toISOString(),
      reviewedBy: reviewerUid
    });
  }
};

window.FB = FB;
window.dispatchEvent(new Event("fb-ready"));

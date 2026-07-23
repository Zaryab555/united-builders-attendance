# United Builders Attendance

Included: Admin/manager/worker roles, PIN login, clock in/out, breaks, GPS, sites, work reports, photo upload, attendance dashboard, worker/site creation and CSV export.

Default PINs:
- Safeer Ahmad: 1001
- Muhammad Kashif Ayoub: 1002
- Zaryab Rashid: 1003
- Other workers: 1004–1014 in listed order.

Setup:
1. Paste your Firebase Web App configuration into `firebase-config.js`.
2. Enable Firestore Database and Firebase Storage.
3. Publish the included Firestore and Storage rules for initial testing.
4. Upload all files to the root of the GitHub repository.

Important: the included Firebase rules are only for testing. Before real employee use, Firebase Authentication and secure role-based rules should be added.

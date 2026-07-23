const CACHE='ub-attendance-v1';
const ASSETS=['./','index.html','style.css','app.js','manifest.json','logo.svg'];
self.addEventListener('install',e=>e.waitUntil(caches.open(CACHE).then(c=>c.addAll(ASSETS))));
self.addEventListener('fetch',e=>e.respondWith(caches.match(e.request).then(r=>r||fetch(e.request))));

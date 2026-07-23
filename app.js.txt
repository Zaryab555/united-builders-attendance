import { firebaseConfig } from "./firebase-config.js";
import { initializeApp } from "https://www.gstatic.com/firebasejs/10.12.5/firebase-app.js";
import { getFirestore, collection, getDocs, doc, setDoc, addDoc, updateDoc, query, where, serverTimestamp, Timestamp } from "https://www.gstatic.com/firebasejs/10.12.5/firebase-firestore.js";
import { getStorage, ref, uploadBytes, getDownloadURL } from "https://www.gstatic.com/firebasejs/10.12.5/firebase-storage.js";

const firebaseApp = initializeApp(firebaseConfig);
const db = getFirestore(firebaseApp);
const storage = getStorage(firebaseApp);
const $ = id => document.getElementById(id);

let currentUser = null, todayAttendance = null, usersCache = [], sitesCache = [];

const defaultUsers = [
{id:"safeer-ahmad",name:"Safeer Ahmad",role:"admin",pin:"1001",active:true},
{id:"muhammad-kashif-ayoub",name:"Muhammad Kashif Ayoub",role:"manager",pin:"1002",active:true},
{id:"zaryab-rashid",name:"Zaryab Rashid",role:"worker",pin:"1003",active:true},
{id:"shahid-ali",name:"Shahid Ali",role:"worker",pin:"1004",active:true},
{id:"amir-shahzad",name:"Amir Shahzad",role:"worker",pin:"1005",active:true},
{id:"gohar-nisar",name:"Gohar Nisar",role:"worker",pin:"1006",active:true},
{id:"nadeem",name:"Nadeem",role:"worker",pin:"1007",active:true},
{id:"mazhar",name:"Mazhar",role:"worker",pin:"1008",active:true},
{id:"faheem",name:"Faheem",role:"worker",pin:"1009",active:true},
{id:"noman",name:"Noman",role:"worker",pin:"1010",active:true},
{id:"shoib",name:"Shoib",role:"worker",pin:"1011",active:true},
{id:"keerus",name:"Keerus",role:"worker",pin:"1012",active:true},
{id:"david",name:"David",role:"worker",pin:"1013",active:true},
{id:"shaban",name:"Shaban",role:"worker",pin:"1014",active:true}
];

const slugify=v=>v.toLowerCase().trim().replace(/[^a-z0-9]+/g,"-").replace(/^-|-$/g,"");
const dateKey=(d=new Date())=>d.toISOString().slice(0,10);
const msg=(id,t)=>$(id).textContent=t;
const formatTime=ts=>!ts?"-":(ts.toDate?ts.toDate():new Date(ts)).toLocaleTimeString([],{hour:"2-digit",minute:"2-digit"});
const durationText=ms=>{const m=Math.max(0,Math.floor(ms/60000));return `${Math.floor(m/60)}h ${m%60}m`;};

async function getLocation(){return new Promise(resolve=>{if(!navigator.geolocation)return resolve(null);navigator.geolocation.getCurrentPosition(p=>resolve({lat:p.coords.latitude,lng:p.coords.longitude,accuracy:p.coords.accuracy}),()=>resolve(null),{enableHighAccuracy:true,timeout:10000});});}

async function seedData(){
  for(const u of defaultUsers) await setDoc(doc(db,"users",u.id),u,{merge:true});
  await setDoc(doc(db,"sites","main-site"),{name:"Main Site",active:true},{merge:true});
}

async function loadUsersAndSites(){
  usersCache=[]; const us=await getDocs(collection(db,"users")); us.forEach(d=>usersCache.push({id:d.id,...d.data()}));
  usersCache=usersCache.filter(x=>x.active!==false).sort((a,b)=>a.name.localeCompare(b.name));
  sitesCache=[]; const ss=await getDocs(collection(db,"sites")); ss.forEach(d=>sitesCache.push({id:d.id,...d.data()}));
  sitesCache=sitesCache.filter(x=>x.active!==false).sort((a,b)=>a.name.localeCompare(b.name));
  $("employeeSelect").innerHTML=usersCache.map(x=>`<option value="${x.id}">${x.name} — ${x.role}</option>`).join("");
  $("siteSelect").innerHTML=sitesCache.map(x=>`<option value="${x.id}">${x.name}</option>`).join("");
}

async function login(){
  const u=usersCache.find(x=>x.id===$("employeeSelect").value);
  if(!u||$("pinInput").value!==u.pin)return msg("loginMessage","Incorrect PIN.");
  currentUser=u; sessionStorage.setItem("ubaUserId",u.id); showDashboard(); await refreshUserData();
}

function showDashboard(){
  $("loginView").classList.add("hidden"); $("dashboardView").classList.remove("hidden");
  $("welcomeTitle").textContent=`Welcome, ${currentUser.name}`; $("roleText").textContent=currentUser.role.toUpperCase();
  if(["admin","manager"].includes(currentUser.role))$("managementPanel").classList.remove("hidden");
  if(currentUser.role==="admin")$("adminOnlyPanel").classList.remove("hidden");
}

async function refreshUserData(){
  const q1=query(collection(db,"attendance"),where("userId","==",currentUser.id),where("dateKey","==",dateKey()));
  const s1=await getDocs(q1); todayAttendance=s1.empty?null:{id:s1.docs[0].id,...s1.docs[0].data()};
  let status="Not clocked in"; if(todayAttendance?.clockIn&&!todayAttendance?.clockOut)status=todayAttendance.breakOpen?"On break":"Clocked in"; if(todayAttendance?.clockOut)status="Clocked out";
  $("currentStatus").textContent=status;
  let t=0;if(todayAttendance?.clockIn){const a=todayAttendance.clockIn.toDate(),b=todayAttendance.clockOut?.toDate()||new Date();t=b-a-(todayAttendance.totalBreakMs||0);} $("todayHours").textContent=durationText(t);
  const start=new Date();start.setDate(start.getDate()-((start.getDay()+6)%7));start.setHours(0,0,0,0);
  const q2=query(collection(db,"attendance"),where("userId","==",currentUser.id),where("clockIn",">=",Timestamp.fromDate(start)));
  const s2=await getDocs(q2);let w=0;s2.forEach(d=>{const a=d.data();if(a.clockIn){w+=(a.clockOut?.toDate()||new Date())-a.clockIn.toDate()-(a.totalBreakMs||0);}});$("weekHours").textContent=durationText(w);
  if(["admin","manager"].includes(currentUser.role))await refreshManagement();
}

async function clockIn(){
  if(todayAttendance?.clockIn)return msg("attendanceMessage","Already clocked in today.");
  const loc=await getLocation(),siteId=$("siteSelect").value,site=sitesCache.find(s=>s.id===siteId);
  await addDoc(collection(db,"attendance"),{userId:currentUser.id,userName:currentUser.name,role:currentUser.role,siteId,siteName:site?.name||"",dateKey:dateKey(),clockIn:serverTimestamp(),clockInLocation:loc,clockOut:null,totalBreakMs:0,breakOpen:false,createdAt:serverTimestamp()});
  msg("attendanceMessage","Clocked in successfully.");await refreshUserData();
}

async function clockOut(){
  if(!todayAttendance?.clockIn||todayAttendance?.clockOut)return msg("attendanceMessage","No active attendance found.");
  const loc=await getLocation();let total=todayAttendance.totalBreakMs||0;if(todayAttendance.breakOpen&&todayAttendance.breakStartedAt)total+=Date.now()-todayAttendance.breakStartedAt.toDate().getTime();
  await updateDoc(doc(db,"attendance",todayAttendance.id),{clockOut:serverTimestamp(),clockOutLocation:loc,breakOpen:false,breakStartedAt:null,totalBreakMs:total});
  msg("attendanceMessage","Clocked out successfully.");await refreshUserData();
}

async function startBreak(){if(!todayAttendance?.clockIn||todayAttendance?.clockOut)return msg("attendanceMessage","Clock in first.");if(todayAttendance.breakOpen)return msg("attendanceMessage","Break is already running.");await updateDoc(doc(db,"attendance",todayAttendance.id),{breakOpen:true,breakStartedAt:serverTimestamp()});msg("attendanceMessage","Break started.");await refreshUserData();}
async function endBreak(){if(!todayAttendance?.breakOpen||!todayAttendance.breakStartedAt)return msg("attendanceMessage","No active break.");const total=(todayAttendance.totalBreakMs||0)+(Date.now()-todayAttendance.breakStartedAt.toDate().getTime());await updateDoc(doc(db,"attendance",todayAttendance.id),{breakOpen:false,breakStartedAt:null,totalBreakMs:total});msg("attendanceMessage","Break ended.");await refreshUserData();}

async function saveReport(){
  const note=$("workNote").value.trim();if(!note)return msg("reportMessage","Please enter work details.");
  let photoUrl="";const file=$("photoInput").files[0];if(file){const r=ref(storage,`reports/${currentUser.id}/${Date.now()}-${file.name}`);await uploadBytes(r,file);photoUrl=await getDownloadURL(r);}
  await addDoc(collection(db,"reports"),{userId:currentUser.id,userName:currentUser.name,siteId:$("siteSelect").value,siteName:sitesCache.find(s=>s.id===$("siteSelect").value)?.name||"",note,photoUrl,dateKey:dateKey(),createdAt:serverTimestamp()});
  $("workNote").value="";$("photoInput").value="";msg("reportMessage","Report saved.");
}

async function refreshManagement(){
  const q=query(collection(db,"attendance"),where("dateKey","==",dateKey()));const s=await getDocs(q);const rows=[];s.forEach(d=>rows.push({id:d.id,...d.data()}));rows.sort((a,b)=>(a.userName||"").localeCompare(b.userName||""));
  $("attendanceTableBody").innerHTML=rows.map(a=>`<tr><td>${a.userName||""}</td><td>${a.role||""}</td><td>${a.siteName||""}</td><td>${formatTime(a.clockIn)}</td><td>${formatTime(a.clockOut)}</td><td>${a.clockOut?"Clocked out":a.breakOpen?"On break":"Working"}</td></tr>`).join("")||`<tr><td colspan="6">No attendance yet.</td></tr>`;
}

async function addWorker(){const name=$("newWorkerName").value.trim(),role=$("newWorkerRole").value,pin=$("newWorkerPin").value.trim();if(!name||!/^\d{4}$/.test(pin))return alert("Enter a name and a 4-digit PIN.");await setDoc(doc(db,"users",slugify(name)||`worker-${Date.now()}`),{name,role,pin,active:true},{merge:true});$("newWorkerName").value="";$("newWorkerPin").value="";await loadUsersAndSites();alert("Worker added.");}
async function addSite(){const name=$("newSiteName").value.trim();if(!name)return alert("Enter a site name.");await setDoc(doc(db,"sites",`${slugify(name)}-${Date.now()}`),{name,active:true});$("newSiteName").value="";await loadUsersAndSites();alert("Site added.");}

async function exportCsv(){const s=await getDocs(collection(db,"attendance"));const lines=[["Name","Role","Date","Site","Clock In","Clock Out","Break Minutes"]];s.forEach(d=>{const a=d.data();lines.push([a.userName||"",a.role||"",a.dateKey||"",a.siteName||"",formatTime(a.clockIn),formatTime(a.clockOut),Math.round((a.totalBreakMs||0)/60000)]);});const csv=lines.map(r=>r.map(v=>`"${String(v).replaceAll('"','""')}"`).join(",")).join("\n");const blob=new Blob([csv],{type:"text/csv"}),url=URL.createObjectURL(blob),a=document.createElement("a");a.href=url;a.download=`united-builders-attendance-${dateKey()}.csv`;a.click();URL.revokeObjectURL(url);}

$("loginBtn")?.addEventListener("click",login);$("logoutBtn")?.addEventListener("click",()=>{sessionStorage.clear();location.reload();});$("clockInBtn")?.addEventListener("click",clockIn);$("clockOutBtn")?.addEventListener("click",clockOut);$("breakStartBtn")?.addEventListener("click",startBreak);$("breakEndBtn")?.addEventListener("click",endBreak);$("saveReportBtn")?.addEventListener("click",saveReport);$("refreshAdminBtn")?.addEventListener("click",refreshManagement);$("exportCsvBtn")?.addEventListener("click",exportCsv);$("addWorkerBtn")?.addEventListener("click",addWorker);$("addSiteBtn")?.addEventListener("click",addSite);

try{msg("loginMessage","Connecting to Firebase...");await seedData();await loadUsersAndSites();msg("loginMessage","");const saved=sessionStorage.getItem("ubaUserId");if(saved){currentUser=usersCache.find(x=>x.id===saved);if(currentUser){showDashboard();await refreshUserData();}}}catch(e){console.error(e);msg("loginMessage","Firebase connection failed. Check firebase-config.js and Firestore rules.");}

const workers=Array.from({length:15},(_,i)=>({id:i+1,name:`Worker ${String(i+1).padStart(2,'0')}`,pin:String(1001+i)}));let user=null;
const $=x=>document.getElementById(x),get=()=>JSON.parse(localStorage.getItem('ub_att')||'[]'),set=x=>localStorage.setItem('ub_att',JSON.stringify(x));
function show(id){['login','worker','admin'].forEach(x=>$(x).classList.add('hidden'));$(id).classList.remove('hidden')}
function login(){let p=$('pin').value;if(p==='9999'){user={admin:true};show('admin');renderAdmin();return}let w=workers.find(x=>x.pin===p);if(!w){$('err').textContent='Incorrect PIN';return}user=w;show('worker');$('wname').textContent=w.name;renderWorker()}
function logout(){user=null;$('pin').value='';show('login')}
function current(id){return get().find(x=>x.workerId===id&&!x.out)}
function mins(a,b){return Math.max(0,Math.round((new Date(b)-new Date(a))/60000))}
function breakM(e){return (e.breaks||[]).reduce((s,b)=>s+(b.end?mins(b.start,b.end):0),0)}
function worked(e){return e.out?Math.max(0,mins(e.in,e.out)-breakM(e)):0}
function fmt(m){return `${Math.floor(m/60)}h ${String(m%60).padStart(2,'0')}m`}
function weekStart(){let d=new Date(),n=(d.getDay()+6)%7;d.setHours(0,0,0,0);d.setDate(d.getDate()-n);return d}
function thisWeek(t){return new Date(t)>=weekStart()}
function geo(){return new Promise(r=>navigator.geolocation?navigator.geolocation.getCurrentPosition(p=>r({lat:p.coords.latitude,lng:p.coords.longitude}),()=>r(null),{timeout:8000}):r(null))}
async function act(type){let all=get(),e=current(user.id),now=new Date().toISOString();if(type==='in'){if(e)return alert('Already clocked in');let loc=await geo();all.push({id:Date.now(),workerId:user.id,site:$('site').value,in:now,out:null,breaks:[],inLoc:loc})}if(type==='breakStart'){if(!e)return alert('Clock in first');if(e.breaks.some(b=>!b.end))return alert('Break already started');e.breaks.push({start:now,end:null})}if(type==='breakEnd'){if(!e)return alert('Clock in first');let b=e.breaks.find(b=>!b.end);if(!b)return alert('No active break');b.end=now}if(type==='out'){if(!e)return alert('Clock in first');let b=e.breaks.find(b=>!b.end);if(b)b.end=now;e.out=now;e.outLoc=await geo()}set(all);renderWorker()}
function renderWorker(){let e=current(user.id),onBreak=e&&e.breaks.some(b=>!b.end);$('status').textContent=onBreak?'Status: On break':e?'Status: Clocked in':'Status: Not clocked in';let total=get().filter(x=>x.workerId===user.id&&thisWeek(x.in)).reduce((s,x)=>s+worked(x),0);$('week').textContent=fmt(total)}
function renderAdmin(){let all=get();$('rows').innerHTML=workers.map(w=>{let e=current(w.id),total=all.filter(x=>x.workerId===w.id&&thisWeek(x.in)).reduce((s,x)=>s+worked(x),0);return `<tr><td>${w.name}</td><td>${e?(e.breaks.some(b=>!b.end)?'On break':'Clocked in'):'Off'}</td><td>${e?e.site:'—'}</td><td>${fmt(total)}</td></tr>`}).join('')}
function exportCSV(){let rows=[['Worker','Site','Clock In','Clock Out','Break Minutes','Worked Minutes','Latitude','Longitude']];get().forEach(e=>{let w=workers.find(x=>x.id===e.workerId);rows.push([w.name,e.site,e.in,e.out||'',breakM(e),worked(e),e.inLoc?.lat||'',e.inLoc?.lng||''])});let csv=rows.map(r=>r.map(v=>`"${String(v).replaceAll('"','""')}"`).join(',')).join('\n');let a=document.createElement('a');a.href=URL.createObjectURL(new Blob([csv],{type:'text/csv'}));a.download='united-builders-attendance.csv';a.click()}

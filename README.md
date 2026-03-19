scriptml>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>LWU Model v5.5</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@300;400;700;800&display=swap" rel="stylesheet">
    <style>
        body { font-family: 'Sarabun', sans-serif; background-color: #f8fafc; -webkit-tap-highlight-color: transparent; }
        /* สไลเดอร์ขนาดใหญ่พิเศษสำหรับนิ้วคนไข้ */
        input[type="range"] { -webkit-appearance: none; width: 100%; height: 20px; background: #e2e8f0; border-radius: 10px; outline: none; margin: 25px 0; }
        input[type="range"]::-webkit-slider-thumb { -webkit-appearance: none; width: 45px; height: 45px; background: #6366f1; border-radius: 50%; border: 6px solid white; box-shadow: 0 5px 15px rgba(0,0,0,0.3); }
        .glass-card { background: white; border-radius: 2.5rem; box-shadow: 0 10px 25px rgba(0,0,0,0.05); border: 1px solid #f1f5f9; padding: 2rem; margin-bottom: 2rem; }
        .btn-psy { flex: 1; padding: 1.2rem 0; font-weight: 800; border-radius: 1.2rem; background-color: #f1f5f9; color: #64748b; font-size: 0.9rem; border: none; transition: all 0.2s; }
        .active-psy { background-color: #6366f1 !important; color: white !important; transform: scale(1.1); box-shadow: 0 5px 15px rgba(99,102,241,0.4); }
    </style>
</head>
<body class="pb-20">
    <header class="bg-indigo-700 text-white pt-12 pb-24 px-6 text-center">
        <h1 class="text-4xl font-black italic tracking-tighter uppercase">LWU Model <span class="text-indigo-300">v5.5</span></h1>
        <p class="text-indigo-200 text-xs mt-2 uppercase tracking-widest font-bold">Palliative Care Support System</p>
        
        <button id="submit-btn" class="mt-10 bg-white text-indigo-700 py-6 px-16 rounded-[2.5rem] text-2xl font-black shadow-2xl active:scale-90 transition-all">
            ยืนยันและส่งข้อมูล
        </button>
        <div id="status" class="mt-4 hidden text-indigo-100 font-bold">กำลังบันทึกข้อมูล...</div>
    </header>

    <main class="max-w-2xl mx-auto px-6 -mt-16">
        <!-- ส่วนที่ 1 -->
        <section class="glass-card">
            <h3 class="font-black text-slate-700 mb-8 border-l-8 border-indigo-600 pl-4 text-xl italic">1. การประเมินปัจจัย (เลื่อนแถบสี)</h3>
            <div id="factors-container" class="space-y-12"></div>
        </section>

        <!-- ส่วนแสดงผล -->
        <section class="glass-card flex flex-col items-center">
            <div class="w-full h-[320px] flex justify-center"><canvas id="radarChart"></canvas></div>
            <div class="text-center mt-6">
                <span class="text-xs font-black text-slate-400 uppercase tracking-widest">คะแนนความคุ้มค่าชีวิต</span>
                <div id="score-display" class="text-8xl font-black text-indigo-600 italic">5.0</div>
            </div>
        </section>

        <!-- ส่วนที่ 2 -->
        <section class="glass-card">
            <h3 class="font-black text-slate-700 mb-8 border-l-8 border-indigo-600 pl-4 text-xl italic">2. สภาวะด้านจิตใจ (กดเลือกตัวเลข)</h3>
            <div id="psy-container" class="space-y-12"></div>
        </section>
    </main>

    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-app.js";
        import { getFirestore, collection, addDoc, serverTimestamp } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-firestore.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/10.8.0/firebase-auth.js";

        const firebaseConfig = {
            apiKey: "AIzaSyD-32kx1r85MtnnB2XIU_IES-CXPlpFzf8",
            authDomain: "the-last-word-utility-model.firebaseapp.com",
            projectId: "the-last-word-utility-model",
            storageBucket: "the-last-word-utility-model.firebasestorage.app",
            messagingSenderId: "563062895044",
            appId: "1:563062895044:web:b18c0c63a9974ef236b491"
        };

        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);
        const appId = "the-last-word-utility-model";

        let factors = { acceptance: 5, regret: 5, pain: 5, wish: 5, family: 5 };
        let psy = { meaning: 5, peace: 5, connection: 5, forgiveness: 5 };
        let chart = null;

        const fLabels = { acceptance: 'การยอมรับสภาวะ', regret: 'ความห่วงกังวล', pain: 'ความเจ็บปวด', wish: 'ความต้องการสุดท้าย', family: 'ความพร้อมของญาติ' };
        const pLabels = { meaning: 'ชีวิตมีจุดหมาย', peace: 'จิตใจสงบนิ่ง', connection: 'ความสัมพันธ์ดี', forgiveness: 'การปล่อยวาง' };

        window.onload = async () => {
            renderControls();
            updateAll();
            await signInAnonymously(auth);
        };

        function renderControls() {
            const fc = document.getElementById('factors-container');
            Object.keys(factors).forEach(key => {
                const div = document.createElement('div');
                div.innerHTML = `
                    <div class="flex justify-between items-end font-black mb-1">
                        <span class="text-slate-500 text-sm uppercase tracking-tighter">${fLabels[key]}</span>
                        <span class="text-indigo-600 text-3xl" id="v-${key}">5</span>
                    </div>
                    <input type="range" min="0" max="10" value="5" class="f-input" data-key="${key}">
                `;
                fc.appendChild(div);
            });

            document.querySelectorAll('.f-input').forEach(input => {
                input.oninput = (e) => {
                    factors[e.target.dataset.key] = parseInt(e.target.value);
                    document.getElementById(`v-${e.target.dataset.key}`).innerText = e.target.value;
                    updateAll();
                };
            });

            const pc = document.getElementById('psy-container');
            Object.keys(psy).forEach(key => {
                const div = document.createElement('div');
                div.innerHTML = `<p class="font-black mb-4 text-slate-400 uppercase text-xs tracking-widest">${pLabels[key]}</p><div class="flex gap-2" id="g-${key}"></div>`;
                pc.appendChild(div);
                const g = document.getElementById(`g-${key}`);
                for(let i=1; i<=10; i++) {
                    const b = document.createElement('button');
                    b.className = `btn-psy ${i===5?'active-psy':''}`;
                    b.innerText = i;
                    b.onclick = () => {
                        psy[key] = i;
                        Array.from(g.children).forEach(x => x.classList.remove('active-psy'));
                        b.classList.add('active-psy');
                    };
                    g.appendChild(b);
                }
            });
        }

        function updateAll() {
            const s = ((factors.acceptance*0.4)+(factors.wish*0.3)+(factors.family*0.3)-(factors.pain*0.5)-(factors.regret*0.3)).toFixed(1);
            document.getElementById('score-display').innerText = s;
            
            const ctx = document.getElementById('radarChart').getContext('2d');
            if(chart) chart.destroy();
            chart = new Chart(ctx, {
                type: 'radar',
                data: {
                    labels: ['ยอมรับ', 'ห่วง', 'เจ็บ', 'ต้องการ', 'ญาติ'],
                    datasets: [{
                        data: Object.values(factors),
                        borderColor: '#6366f1',
                        backgroundColor: 'rgba(99, 102, 241, 0.1)',
                        borderWidth: 4,
                        pointBackgroundColor: '#fff',
                        pointBorderColor: '#6366f1',
                        pointRadius: 6
                    }]
                },
                options: {
                    scales: { r: { min: 0, max: 10, ticks: { display: false }, grid: { color: '#f1f5f9' } } },
                    plugins: { legend: { display: false } }
                }
            });
        }

        document.getElementById('submit-btn').onclick = async () => {
            const btn = document.getElementById('submit-btn');
            const st = document.getElementById('status');
            btn.disabled = true;
            st.classList.remove('hidden');
            
            try {
                await addDoc(collection(db, 'artifacts', appId, 'public', 'data', 'submissions'), {
                    factors, psy, score: document.getElementById('score-display').innerText,
                    timestamp: serverTimestamp(),
                    userId: auth.currentUser?.uid || "anon"
                });
                st.innerText = "✓ บันทึกข้อมูลเรียบร้อยแล้ว ขอบคุณครับ";
                btn.innerText = "ส่งสำเร็จ";
                btn.className = "mt-10 bg-emerald-500 text-white py-6 px-16 rounded-[2.5rem] text-2xl font-black shadow-lg";
            } catch (e) {
                alert("ล้มเหลว: " + e.message);
                btn.disabled = false;
                st.classList.add('hidden');
            }
        };
    </script>
</body>
</html>


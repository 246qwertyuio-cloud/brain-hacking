(function(){
    // ========== 러브 엔진 - 궁극 확장판 (Ultimate) ==========
    console.clear();
    
    // ---------- 사용자 설정 ----------
    let userName = "혜림";      // 변경 가능 (setName())
    let loveSpeed = 20;         // ms (setSpeed())
    let speechEnabled = false;
    
    // ---------- 기본 상수 ----------
    const HEX = '0123456789abcdef';
    const LOVE = ['사랑해', userName, '무한', '💖', '∞', 'LOVE', '愛', '영원히', '님', '소중한', '그대', '반쪽', '운명', '내 사람', '보물'];
    const COLORS = ['#ff3366', '#ff6699', '#ff99cc', '#ff66b5', '#ff1493', '#ff6eb4', '#ff4081', '#f50057'];
    let count = 0;
    let strength = 0;
    let opsCount = 0;
    let isOverloaded = false;
    let intervals = [];
    let audioCtx = null;
    let soundEnabled = false;
    
    // ---------- DOM 요소 ----------
    let canvas = null, ctx = null;
    let particles = [];
    let floatingHeart = null;
    let progressBar = null;
    let controlPanel = null;      // 드래그 가능한 제어판
    let graphCanvas = null;       // 심박수 그래프
    let graphCtx = null;
    let bpmHistory = [];
    
    // ---------- Three.js 3D 하트 ----------
    let scene, camera, renderer, heartMesh;
    let threeLoaded = false;
    
    // ---------- 유틸 ----------
    function hex(n) { let s=''; for(let i=0;i<n;i++) s+=HEX[Math.floor(Math.random()*16)]; return s; }
    function rnd(a,b) { return Math.floor(Math.random()*(b-a+1)+a); }
    function randomColor() { return COLORS[Math.floor(Math.random()*COLORS.length)]; }
    function rainbowLog(...args) { console.log(`%c${args.join(' ')}`, `color: hsl(${Math.random()*360},100%,65%);font-weight:bold;`); }
    
    // LOVE 배열 업데이트 (이름 변경 시)
    function updateLoveArray() {
        LOVE[1] = userName;
    }
    
    // ---------- Web Speech (말하기) ----------
    function speakLove() {
        if (!speechEnabled) return;
        const msg = new SpeechSynthesisUtterance(`${userName} 사랑해, 영원히`);
        msg.lang = 'ko-KR';
        msg.rate = 1.0;
        msg.pitch = 1.2;
        window.speechSynthesis.cancel();
        window.speechSynthesis.speak(msg);
    }
    
    // ---------- 3D 하트 (Three.js) ----------
    function init3DHeart() {
        if (threeLoaded) return;
        const script = document.createElement('script');
        script.src = 'https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js';
        script.onload = () => {
            const container = document.createElement('div');
            container.style.position = 'fixed';
            container.style.bottom = '20px';
            container.style.left = '20px';
            container.style.width = '180px';
            container.style.height = '180px';
            container.style.zIndex = '10010';
            container.style.pointerEvents = 'none';
            document.body.appendChild(container);
            
            scene = new THREE.Scene();
            camera = new THREE.PerspectiveCamera(45, 1, 0.1, 1000);
            camera.position.z = 2;
            renderer = new THREE.WebGLRenderer({ alpha: true });
            renderer.setSize(180, 180);
            container.appendChild(renderer.domElement);
            
            const geometry = new THREE.SphereGeometry(0.5, 32, 32);
            const material = new THREE.MeshStandardMaterial({ color: 0xff3366, emissive: 0x441122, roughness: 0.3 });
            heartMesh = new THREE.Mesh(geometry, material);
            // 하트 모양 만들기 위해 그룹으로 변형 (간단한 구를 늘림)
            heartMesh.scale.set(0.8, 0.9, 0.6);
            const group = new THREE.Group();
            const sphere2 = new THREE.Mesh(geometry, material);
            sphere2.position.set(0.5, 0.2, 0);
            sphere2.scale.set(0.7, 0.8, 0.6);
            const sphere3 = new THREE.Mesh(geometry, material);
            sphere3.position.set(-0.5, 0.2, 0);
            sphere3.scale.set(0.7, 0.8, 0.6);
            group.add(heartMesh);
            group.add(sphere2);
            group.add(sphere3);
            scene.add(group);
            
            const light = new THREE.DirectionalLight(0xffffff, 1);
            light.position.set(1, 2, 3);
            scene.add(light);
            const ambient = new THREE.AmbientLight(0x442222);
            scene.add(ambient);
            
            function animate3D() {
                if (!group) return;
                group.rotation.y += 0.01;
                group.rotation.x = Math.sin(Date.now() * 0.003) * 0.2;
                renderer.render(scene, camera);
                requestAnimationFrame(animate3D);
            }
            animate3D();
            threeLoaded = true;
        };
        document.head.appendChild(script);
    }
    
    // ---------- 심박수 그래프 ----------
    function initGraph() {
        graphCanvas = document.createElement('canvas');
        graphCanvas.width = 300;
        graphCanvas.height = 100;
        graphCanvas.style.position = 'fixed';
        graphCanvas.style.bottom = '10px';
        graphCanvas.style.right = '10px';
        graphCanvas.style.zIndex = '10011';
        graphCanvas.style.backgroundColor = 'rgba(0,0,0,0.6)';
        graphCanvas.style.borderRadius = '10px';
        graphCanvas.style.padding = '5px';
        graphCanvas.style.border = '1px solid #ff6699';
        document.body.appendChild(graphCanvas);
        graphCtx = graphCanvas.getContext('2d');
        
        setInterval(() => {
            let bpm = 60 + Math.floor(strength / 1.5) + (isOverloaded ? 40 : 0);
            bpmHistory.push(bpm);
            if (bpmHistory.length > 50) bpmHistory.shift();
            if (!graphCtx) return;
            graphCtx.clearRect(0, 0, graphCanvas.width, graphCanvas.height);
            graphCtx.strokeStyle = '#ff6699';
            graphCtx.lineWidth = 2;
            graphCtx.beginPath();
            for (let i = 0; i < bpmHistory.length; i++) {
                let x = (i / bpmHistory.length) * graphCanvas.width;
                let y = graphCanvas.height - (bpmHistory[i] / 150) * graphCanvas.height;
                if (i === 0) graphCtx.moveTo(x, y);
                else graphCtx.lineTo(x, y);
            }
            graphCtx.stroke();
            graphCtx.fillStyle = '#ff99cc';
            graphCtx.font = '10px sans-serif';
            graphCtx.fillText(`❤️ BPM: ${bpmHistory[bpmHistory.length-1] || 70}`, 5, 15);
        }, 1000);
    }
    
    // ---------- 드래그 가능한 제어판 ----------
    function initControlPanel() {
        controlPanel = document.createElement('div');
        controlPanel.innerHTML = `
            <div style="background:#ff3366; padding:6px; cursor:move; text-align:center; color:white; border-radius:10px 10px 0 0;">❤️ 러브 컨트롤 ❤️</div>
            <div style="padding:10px; background:rgba(255,240,245,0.95); border-radius:0 0 10px 10px;">
                <button id="btnStop">⏹️ 정지</button>
                <button id="btnStart">▶️ 재시작</button>
                <button id="btnConfetti">🎉 폭죽</button>
                <button id="btnLetter">💌 편지</button>
                <button id="btnSound">🔊 소리 On/Off</button>
                <button id="btnSpeak">🗣️ 말하기 On/Off</button>
                <input type="text" id="nameInput" placeholder="이름 입력" value="${userName}" style="margin-top:8px; width:90%;">
                <button id="btnSetName">💖 이름 변경</button>
                <div style="font-size:12px; margin-top:8px;">❤️ 강도: <span id="strengthDisplay">0</span>%</div>
            </div>
        `;
        controlPanel.style.position = 'fixed';
        controlPanel.style.top = '80px';
        controlPanel.style.left = '20px';
        controlPanel.style.width = '200px';
        controlPanel.style.zIndex = '20000';
        controlPanel.style.backgroundColor = 'white';
        controlPanel.style.borderRadius = '12px';
        controlPanel.style.boxShadow = '0 0 20px rgba(255,105,180,0.5)';
        controlPanel.style.cursor = 'default';
        document.body.appendChild(controlPanel);
        
        // 드래그 기능
        let drag = false, offsetX, offsetY;
        controlPanel.querySelector('div:first-child').addEventListener('mousedown', (e) => {
            drag = true;
            offsetX = e.clientX - controlPanel.offsetLeft;
            offsetY = e.clientY - controlPanel.offsetTop;
        });
        document.addEventListener('mousemove', (e) => {
            if (drag) {
                controlPanel.style.left = (e.clientX - offsetX) + 'px';
                controlPanel.style.top = (e.clientY - offsetY) + 'px';
            }
        });
        document.addEventListener('mouseup', () => drag = false);
        
        // 버튼 이벤트
        document.getElementById('btnStop').onclick = () => window.stopLoveMachine();
        document.getElementById('btnStart').onclick = () => window.startLoveMachine();
        document.getElementById('btnConfetti').onclick = () => window.loveConfetti();
        document.getElementById('btnLetter').onclick = () => window.showLetter();
        document.getElementById('btnSound').onclick = () => window.toggleSound();
        document.getElementById('btnSpeak').onclick = () => { speechEnabled = !speechEnabled; console.log(`%c🗣️ 말하기 ${speechEnabled ? "ON" : "OFF"}`, "color:#ff66aa"); };
        document.getElementById('btnSetName').onclick = () => {
            let newName = document.getElementById('nameInput').value.trim();
            if (newName) { userName = newName; updateLoveArray(); window.showLoveMessage(`💖 이름이 ${userName}(으)로 변경됨 💖`); speakLove(); }
        };
        
        setInterval(() => {
            const span = document.getElementById('strengthDisplay');
            if (span) span.innerText = Math.floor(strength);
        }, 200);
    }
    
    // ---------- 마우스 클릭 하트 폭발 ----------
    function initClickExplosion() {
        document.addEventListener('click', (e) => {
            for (let i=0; i<12; i++) {
                setTimeout(() => {
                    const heart = document.createElement('div');
                    heart.innerHTML = '❤️';
                    heart.style.position = 'fixed';
                    heart.style.left = e.clientX + 'px';
                    heart.style.top = e.clientY + 'px';
                    heart.style.fontSize = rnd(15,35) + 'px';
                    heart.style.pointerEvents = 'none';
                    heart.style.zIndex = '20001';
                    heart.style.opacity = '1';
                    heart.style.transition = 'all 0.6s ease-out';
                    document.body.appendChild(heart);
                    const dx = (Math.random() - 0.5) * 100;
                    const dy = (Math.random() - 1) * 150;
                    heart.style.transform = `translate(${dx}px, ${dy}px) rotate(${rnd(0,360)}deg)`;
                    heart.style.opacity = '0';
                    setTimeout(() => heart.remove(), 600);
                }, i*20);
            }
        });
    }
    
    // ---------- 기존 코어 기능 ----------
    function safeAddLog() {
        try {
            const msgList = [
                () => `[ENC] "${LOVE[rnd(0, LOVE.length-1)]}" → 0x${hex(8)}...${hex(4)}`,
                () => `[KEY] RSA-${rnd(8,64)*128}bit: ${hex(16)}...`,
                () => `[SHA-💖] ${hex(12)} → ${hex(32)}`,
                () => `[AES-∞] 블록 #${rnd(1000,99999)} 완료`,
                () => `[VERIFY] 무결성: ✓ PASS`,
                () => `[ITER-${count}] ${LOVE[rnd(0, LOVE.length-1)]} — ∞`,
                () => `[BPM] 심박수 ${60+Math.floor(strength/1.5)}`,
                () => `[3D] 하트 회전 중...`
            ];
            const msg = msgList[rnd(0, msgList.length-1)]();
            const now = new Date();
            const ts = `${now.getHours().toString().padStart(2,'0')}:${now.getMinutes().toString().padStart(2,'0')}:${now.getSeconds().toString().padStart(2,'0')}.${now.getMilliseconds().toString().padStart(3,'0')}`;
            console.log(`%c[${ts}] 💕 ${msg}`, `color: ${randomColor()}; font-weight: bold;`);
            count++;
            opsCount++;
            if (count % 100 === 0) window.loveConfetti?.();
            if (count % 250 === 0 && speechEnabled) speakLove();
        } catch(e) {}
    }
    
    function startCore() {
        const id = setInterval(() => {
            if (!isOverloaded) {
                const burst = Math.random() < 0.45 ? rnd(5,9) : 3;
                for (let i=0;i<burst;i++) safeAddLog();
                rainbowLog(`💎 HASH: ${hex(64)}`);
            } else {
                for (let i=0;i<12;i++) safeAddLog();
            }
        }, loveSpeed);
        intervals.push(id);
    }
    
    function startStat() {
        const id = setInterval(() => {
            const percent = strength.toFixed(1);
            console.log(`\n%c[❤️ STAT] ${opsCount} ops/sec | 총 ${count} | 강도 ${percent}%`, `color: ${strength>=100?'#ff0066':'#ff99cc'}; font-weight:bold;`);
            opsCount = 0;
            if (!isOverloaded) {
                if (strength < 100) strength = Math.min(100, strength + rnd(7,15));
                else triggerOverload();
            }
        }, 800);
        intervals.push(id);
    }
    
    function triggerOverload() {
        if (isOverloaded) return;
        isOverloaded = true;
        console.log("%c🔥💖 LOVE OVERLOAD ∞ - 궁극 증식 🔥💖", "color:#ff0066; font-size:26px;");
        const overloadId = setInterval(() => { for(let i=0;i<15;i++) safeAddLog(); }, 8);
        intervals.push(overloadId);
        window.loveConfetti();
        setInterval(() => { if(isOverloaded) window.loveConfetti(); }, 4000);
        if (floatingHeart) floatingHeart.style.animation = 'pulse 0.4s infinite';
    }
    
    // ---------- Confetti 래퍼 ----------
    function loadConfettiLib() {
        if (typeof window.confetti !== 'undefined') return;
        const script = document.createElement('script');
        script.src = 'https://cdn.jsdelivr.net/npm/canvas-confetti@1';
        document.head.appendChild(script);
    }
    window.loveConfetti = function() {
        if (typeof window.confetti === 'function') {
            window.confetti({ particleCount: 200, spread: 100, origin: { y: 0.6 }, colors: COLORS });
            window.confetti({ particleCount: 80, angle: 60, spread: 60 });
            window.confetti({ particleCount: 80, angle: 120, spread: 60 });
        } else console.log("%c🎊 confetti 로딩 중...", "color:#ff99cc");
    };
    
    // ---------- 플로팅 메시지 (기존 확장 호환) ----------
    let showLoveMessageFn;
    function initFloatingMessageCompat() {
        const div = document.createElement('div');
        div.style.position = 'fixed';
        div.style.bottom = '80px';
        div.style.right = '20px';
        div.style.backgroundColor = '#ff6699cc';
        div.style.color = 'white';
        div.style.padding = '6px 15px';
        div.style.borderRadius = '30px';
        div.style.zIndex = '10012';
        div.style.transition = 'opacity 0.3s';
        div.style.opacity = '0';
        document.body.appendChild(div);
        window.showLoveMessage = (msg) => {
            div.innerText = msg;
            div.style.opacity = '1';
            setTimeout(() => div.style.opacity = '0', 1200);
        };
    }
    
    // ---------- 기존 파티클, 마우스 하트 등 재사용 (간략화) ----------
    function initCanvasParticles() {
        canvas = document.createElement('canvas');
        canvas.style.position = 'fixed';
        canvas.style.top = 0;
        canvas.style.left = 0;
        canvas.style.width = '100%';
        canvas.style.height = '100%';
        canvas.style.pointerEvents = 'none';
        canvas.style.zIndex = '9998';
        document.body.appendChild(canvas);
        ctx = canvas.getContext('2d');
        const resize = () => { canvas.width = window.innerWidth; canvas.height = window.innerHeight; };
        window.addEventListener('resize', resize);
        resize();
        class HeartParticle {
            constructor() {
                this.x = Math.random() * canvas.width;
                this.y = Math.random() * canvas.height;
                this.size = rnd(12,28);
                this.speedY = rnd(3,10)/10;
                this.speedX = (Math.random()-0.5)*0.8;
                this.opacity = Math.random()*0.6+0.2;
                this.color = `hsla(${Math.random()*60+330},100%,65%,${this.opacity})`;
            }
            update() {
                this.y -= this.speedY;
                this.x += this.speedX;
                if(this.y<-40) this.y=canvas.height+40;
                if(this.x<-40) this.x=canvas.width+40;
                if(this.x>canvas.width+40) this.x=-40;
                if(isOverloaded) this.speedY = rnd(6,15)/10;
            }
            draw() { if(ctx) { ctx.font=`${this.size}px "Segoe UI Emoji"`; ctx.fillStyle=this.color; ctx.fillText('❤️',this.x,this.y); } }
        }
        for(let i=0;i<60;i++) particles.push(new HeartParticle());
        function animate() { if(!ctx) return; ctx.clearRect(0,0,canvas.width,canvas.height); particles.forEach(p=>{p.update(); p.draw();}); requestAnimationFrame(animate); }
        animate();
    }
    
    function initFloatingHeart() {
        floatingHeart = document.createElement('div');
        floatingHeart.innerHTML = '💖';
        floatingHeart.style.position = 'fixed';
        floatingHeart.style.fontSize = '45px';
        floatingHeart.style.pointerEvents = 'none';
        floatingHeart.style.zIndex = '10013';
        floatingHeart.style.filter = 'drop-shadow(0 0 5px magenta)';
        document.body.appendChild(floatingHeart);
        document.addEventListener('mousemove', (e) => {
            floatingHeart.style.left = (e.clientX - 25) + 'px';
            floatingHeart.style.top = (e.clientY - 45) + 'px';
            floatingHeart.style.transform = isOverloaded ? 'scale(1.3)' : 'scale(1)';
        });
    }
    
    // ---------- Audio & Sound ----------
    function initAudioSystem() {
        audioCtx = new (window.AudioContext || window.webkitAudioContext)();
        const playBeep = () => {
            if(!soundEnabled || !audioCtx) return;
            const osc = audioCtx.createOscillator();
            const gain = audioCtx.createGain();
            osc.connect(gain);
            gain.connect(audioCtx.destination);
            osc.frequency.value = 100;
            gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
            gain.gain.exponentialRampToValueAtTime(0.0001, audioCtx.currentTime+0.4);
            osc.start();
            osc.stop(audioCtx.currentTime+0.4);
        };
        setInterval(() => { if(soundEnabled && (isOverloaded || Math.random()<0.3)) playBeep(); }, 1800);
        window.toggleSound = () => {
            if(!audioCtx) initAudioSystem();
            soundEnabled = !soundEnabled;
            if(soundEnabled && audioCtx.state === 'suspended') audioCtx.resume();
            console.log(`%c🔊 사운드 ${soundEnabled?"ON":"OFF"}`, "color:#ff66aa");
        };
        document.addEventListener('click', () => { if(audioCtx && audioCtx.state === 'suspended') audioCtx.resume(); }, { once: true });
    }
    
    // ---------- 초기화 ----------
    window.stopLoveMachine = function() {
        intervals.forEach(id=>clearInterval(id));
        intervals = [];
        console.log("%c💔 정지됨. 재시작: startLoveMachine()", "color:#ff6699");
    };
    window.startLoveMachine = function() {
        if(intervals.length) window.stopLoveMachine();
        count=0; strength=0; opsCount=0; isOverloaded=false;
        startCore(); startCore(); startStat();
        console.log("%c💖 재가동", "color:#ff3366");
    };
    window.showLetter = function() {
        const modal = document.createElement('div');
        modal.innerHTML = `<div style="background:white; padding:20px; border-radius:30px; text-align:center;"><h3>💌 러브레터</h3><p>${userName}님, 영원히 사랑해요 💖</p><button onclick="this.parentElement.parentElement.remove()">❤️ 닫기</button></div>`;
        modal.style.position='fixed'; modal.style.top='50%'; modal.style.left='50%'; modal.style.transform='translate(-50%,-50%)'; modal.style.zIndex='30000'; modal.style.backgroundColor='rgba(0,0,0,0.5)'; modal.style.padding='20px'; modal.style.borderRadius='40px';
        document.body.appendChild(modal);
    };
    window.loveStatus = () => console.log(`반복:${count} 강도:${strength}% 과부하:${isOverloaded}`);
    
    // 실행
    function init() {
        console.log("%c🌸🌸🌸 궁극 러브 엔진 - Ultimate Edition 🌸🌸🌸", "color:#ff66aa; font-size:20px;");
        initCanvasParticles();
        initFloatingHeart();
        initFloatingMessageCompat();
        initControlPanel();
        initGraph();
        init3DHeart();
        initClickExplosion();
        initAudioSystem();
        loadConfettiLib();
        startCore(); startCore(); startStat();
        console.log("%c✨ 명령어: stopLoveMachine() | startLoveMachine() | loveStatus() | loveConfetti() | showLetter() | toggleSound()", "color:#aaa");
    }
    init();
})();

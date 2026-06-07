# merry-christmas

<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>平安夜 - 专属定制</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: 'Times New Roman', serif; }
        
        /* 画布全屏 */
        #canvas-container { width: 100vw; height: 100vh; position: absolute; top: 0; left: 0; z-index: 1; }
        
        /* 摄像头小窗 (左下角) */
        #webcam {
            position: absolute;
            bottom: 20px;
            left: 20px;
            width: 200px; /* 调整大小 */
            height: 150px;
            object-fit: cover;
            z-index: 20; /* 在最上层 */
            border-radius: 12px;
            border: 2px solid rgba(212, 175, 55, 0.5); /* 金色边框 */
            box-shadow: 0 0 15px rgba(212, 175, 55, 0.3);
            transform: scaleX(-1); /* 镜像翻转，方便操作 */
            opacity: 0.8;
            transition: opacity 0.3s;
        }
        #webcam:hover { opacity: 1; }

        /* UI 层 */
        #ui-layer {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            z-index: 10; pointer-events: none;
            display: flex; flex-direction: column; align-items: center;
            padding-top: 40px; box-sizing: border-box;
        }
        
        .ui-hidden { opacity: 0; pointer-events: none !important; }

        /* 加载动画 */
        #loader {
            position: absolute; top: 0; left: 0; width: 100%; height: 100%;
            background: #000; z-index: 100;
            display: flex; flex-direction: column; align-items: center; justify-content: center;
            transition: opacity 0.8s ease-out;
        }
        .loader-text {
            color: #d4af37; font-size: 14px; letter-spacing: 4px; margin-top: 20px;
            text-transform: uppercase; font-weight: 100;
        }
        .spinner {
            width: 40px; height: 40px; border: 1px solid rgba(212, 175, 55, 0.2); 
            border-top: 1px solid #d4af37; border-radius: 50%; 
            animation: spin 1s linear infinite;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

        /* 标题样式 */
        h1 { 
            color: #fceea7; font-size: 40px; margin: 0; font-weight: 400; 
            letter-spacing: 2px; 
            text-shadow: 0 0 20px rgba(252, 238, 167, 0.6); 
            font-family: 'Cinzel', serif; opacity: 0.8;
        }

        /* 按钮与提示 */
        .upload-wrapper { margin-top: 15px; pointer-events: auto; text-align: center; }
        .upload-btn {
            background: rgba(20, 20, 20, 0.6); border: 1px solid rgba(212, 175, 55, 0.4); 
            color: #d4af37; padding: 8px 20px; cursor: pointer; text-transform: uppercase; 
            letter-spacing: 2px; font-size: 10px; transition: all 0.4s; display: inline-block;
            backdrop-filter: blur(5px);
        }
        .upload-btn:hover { background: #d4af37; color: #000; box-shadow: 0 0 20px rgba(212, 175, 55, 0.5); }
        .hint-text { color: rgba(255, 255, 255, 0.6); font-size: 10px; margin-top: 10px; letter-spacing: 1px; }

        #file-input { display: none; }
    </style>
    
    <style>@import url('https://fonts.googleapis.com/css2?family=Cinzel:wght@400;700&display=swap');</style>

    <script type="importmap">
        {
            "imports": {
                "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
                "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/",
                "@mediapipe/tasks-vision": "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/+esm"
            }
        }
    </script>
</head>
<body>

    <div id="loader">
        <div class="spinner"></div>
        <div class="loader-text">为刘慧琦准备惊喜中...</div>
    </div>

    <div id="canvas-container"></div>

    <!-- 视频源：现在显示在左下角 -->
    <video id="webcam" autoplay playsinline muted></video>

    <div id="ui-layer">
        <h1>Merry Christmas</h1>
        <div class="upload-wrapper">
            <label class="upload-btn">
                添加照片
                <input type="file" id="file-input" multiple accept="image/*">
            </label>
            <div class="hint-text">对着左下角摄像头：张开手掌查看祝福 | 握拳聚合</div>
        </div>
    </div>

    <script type="module">
        import * as THREE from 'three';
        import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
        import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
        import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
        import { RoomEnvironment } from 'three/addons/environments/RoomEnvironment.js'; 
        import { FilesetResolver, HandLandmarker } from '@mediapipe/tasks-vision';

        // --- CONFIGURATION ---
        const CONFIG = {
            colors: {
                champagneGold: 0xffd966, 
                deepGreen: 0x03180a,     
                accentRed: 0x990000,     
            },
            particles: {
                count: 1200,     // 树的基础粒子
                dustCount: 1500, 
                treeHeight: 22,  
                treeRadius: 7.5    
            },
            textMessage: [
                { text: "平安夜", size: 60, y: 5 },
                { text: "祝刘慧琦平平安安", size: 40, y: -2 },
                { text: "不止今夜", size: 30, y: -7 }
            ]
        };

        const STATE = {
            mode: 'TREE', // TREE, SCATTER (Text), FOCUS
            focusTarget: null,
            hand: { detected: false, x: 0, y: 0 },
            rotation: { x: 0, y: 0 } 
        };

        let scene, camera, renderer, composer;
        let mainGroup; 
        let clock = new THREE.Clock();
        let particleSystem = []; 
        let photoMeshGroup = new THREE.Group();
        let handLandmarker, video;
        let caneTexture; 
        let goldMat, redMat;

        async function init() {
            initThree();
            setupLights();
            createTextures();
            initMaterials();
            createParticles(); 
            createDust();     
            createDefaultPhotos();
            createTextParticles();
            setupPostProcessing();
            setupEvents();
            await initMediaPipe();
            
            const loader = document.getElementById('loader');
            loader.style.opacity = 0;
            setTimeout(() => loader.remove(), 800);

            animate();
        }

        function initThree() {
            const container = document.getElementById('canvas-container');
            scene = new THREE.Scene();
            // 纯黑背景，突出主体
            scene.background = new THREE.Color(0x050505); 
            // 增加一点环境雾
            scene.fog = new THREE.FogExp2(0x050505, 0.02);

            camera = new THREE.PerspectiveCamera(42, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 1, 45); 

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            renderer.toneMapping = THREE.ReinhardToneMapping; 
            renderer.toneMappingExposure = 2.0; 
            container.appendChild(renderer.domElement);

            mainGroup = new THREE.Group();
            scene.add(mainGroup);
        }

        function setupLights() {
            const ambient = new THREE.AmbientLight(0xffffff, 0.4);
            scene.add(ambient);

            const spotGold = new THREE.SpotLight(0xffcc66, 800);
            spotGold.position.set(20, 30, 40);
            scene.add(spotGold);

            const fill = new THREE.DirectionalLight(0xffeebb, 0.8);
            fill.position.set(0, 0, 50);
            scene.add(fill);
        }

        function setupPostProcessing() {
            const renderScene = new RenderPass(scene, camera);
            const bloomPass = new UnrealBloomPass(new THREE.Vector2(window.innerWidth, window.innerHeight), 1.5, 0.4, 0.85);
            bloomPass.threshold = 0.5; 
            bloomPass.strength = 0.6; // 加强一点辉光
            bloomPass.radius = 0.5;

            composer = new EffectComposer(renderer);
            composer.addPass(renderScene);
            composer.addPass(bloomPass);
        }

        function createTextures() {
            const canvas = document.createElement('canvas');
            canvas.width = 128; canvas.height = 128;
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = '#ffffff'; ctx.fillRect(0,0,128,128);
            ctx.fillStyle = '#880000'; 
            ctx.beginPath();
            for(let i=-128; i<256; i+=32) {
                ctx.moveTo(i, 0); ctx.lineTo(i+32, 128); ctx.lineTo(i+16, 128); ctx.lineTo(i-16, 0);
            }
            ctx.fill();
            caneTexture = new THREE.CanvasTexture(canvas);
            caneTexture.wrapS = THREE.RepeatWrapping; caneTexture.wrapT = THREE.RepeatWrapping;
            caneTexture.repeat.set(3, 3);
        }

        function initMaterials() {
            goldMat = new THREE.MeshStandardMaterial({
                color: CONFIG.colors.champagneGold,
                metalness: 0.9, roughness: 0.1,
                emissive: 0x443300, emissiveIntensity: 0.2
            });
            redMat = new THREE.MeshPhysicalMaterial({
                color: CONFIG.colors.accentRed,
                metalness: 0.4, roughness: 0.2, clearcoat: 1.0,
                emissive: 0x440000, emissiveIntensity: 0.4
            });
        }

        class Particle {
            constructor(mesh, type, isDust = false, textPos = null) {
                this.mesh = mesh;
                this.type = type; 
                this.isDust = isDust;
                this.posTree = new THREE.Vector3();
                this.posScatter = new THREE.Vector3();
                this.baseScale = mesh.scale.x; 
                this.targetTextPos = textPos; 
                this.calculatePositions();
            }

            calculatePositions() {
                // TREE
                const h = CONFIG.particles.treeHeight;
                const halfH = h / 2;
                let t = Math.random(); 
                t = Math.pow(t, 0.8); 
                const y = (t * h) - halfH - 2; 
                let rMax = CONFIG.particles.treeRadius * (1.0 - t); 
                if (rMax < 0.2) rMax = 0.2;
                const angle = t * 45 * Math.PI + Math.random() * Math.PI * 2; 
                const r = rMax * (0.5 + Math.random() * 0.5); 
                this.posTree.set(Math.cos(angle) * r, y, Math.sin(angle) * r);

                // SCATTER (TEXT or RANDOM)
                if (this.type === 'TEXT' && this.targetTextPos) {
                    this.posScatter.copy(this.targetTextPos);
                } else {
                    let rScatter = this.isDust ? (15 + Math.random()*20) : (10 + Math.random()*15);
                    const theta = Math.random() * Math.PI * 2;
                    const phi = Math.acos(2 * Math.random() - 1);
                    this.posScatter.set(
                        rScatter * Math.sin(phi) * Math.cos(theta),
                        rScatter * Math.sin(phi) * Math.sin(theta),
                        rScatter * Math.cos(phi)
                    );
                    if (!this.isDust && this.type !== 'PHOTO') {
                        this.posScatter.multiplyScalar(1.5);
                    }
                }
            }

            update(dt, mode, focusTargetMesh) {
                let target = this.posTree;
                let rotate = true;
                
                if (mode === 'SCATTER') {
                    target = this.posScatter;
                    if (this.type === 'TEXT') rotate = false;
                }
                else if (mode === 'FOCUS') {
                    if (this.mesh === focusTargetMesh) {
                        const desiredWorldPos = new THREE.Vector3(0, 1, 38);
                        const invMatrix = new THREE.Matrix4().copy(mainGroup.matrixWorld).invert();
                        target = desiredWorldPos.applyMatrix4(invMatrix);
                        rotate = false;
                    } else {
                        target = this.posScatter;
                    }
                }

                const lerpSpeed = (mode === 'FOCUS' && this.mesh === focusTargetMesh) ? 4.0 : 2.5; 
                this.mesh.position.lerp(target, lerpSpeed * dt);

                if (rotate && mode === 'SCATTER') {
                    this.mesh.rotation.x += dt * 0.5;
                    this.mesh.rotation.y += dt * 0.5;
                } else if (mode === 'TREE') {
                    this.mesh.rotation.set(0,0,0); 
                    this.mesh.rotation.y = Math.atan2(this.mesh.position.x, this.mesh.position.z);
                } else if (this.type === 'TEXT' && mode === 'SCATTER') {
                    this.mesh.rotation.set(0,0,0);
                }

                if (mode === 'FOCUS' && this.mesh === focusTargetMesh) {
                    this.mesh.lookAt(camera.position); 
                }

                let s = this.baseScale;
                if (this.isDust) {
                    s = this.baseScale * (0.8 + 0.4 * Math.sin(clock.elapsedTime * 4 + this.mesh.id));
                    if (mode === 'TREE') s = 0; 
                } else if (mode === 'SCATTER' && this.type === 'PHOTO') {
                    s = this.baseScale * 2.0; 
                } else if (mode === 'FOCUS') {
                    if (this.mesh === focusTargetMesh) s = 4.5; 
                    else s = 0.1; 
                }
                this.mesh.scale.lerp(new THREE.Vector3(s,s,s), 4*dt);
            }
        }

        function createTextParticles() {
            const canvas = document.createElement('canvas');
            const ctx = canvas.getContext('2d');
            const size = 1024; 
            canvas.width = size;
            canvas.height = size;

            ctx.fillStyle = '#000000';
            ctx.fillRect(0, 0, size, size);
            ctx.textAlign = 'center';
            ctx.textBaseline = 'middle';
            ctx.fillStyle = '#ffffff'; 

            CONFIG.textMessage.forEach(item => {
                const fontSize = item.size * 2.5; 
                ctx.font = `bold ${fontSize}px "Microsoft YaHei", "SimHei", sans-serif`;
                ctx.fillText(item.text, size / 2, size / 2 + item.y * 30);
            });

            const imageData = ctx.getImageData(0, 0, size, size);
            const data = imageData.data;
            const sphereGeo = new THREE.SphereGeometry(0.3, 16, 16); 
            const step = 6; 
            
            for (let y = 0; y < size; y += step) {
                for (let x = 0; x < size; x += step) {
                    const alpha = data[(y * size + x) * 4]; 
                    if (alpha > 128) {
                        const posX = (x - size / 2) / 25; 
                        const posY = -(y - size / 2) / 25; 
                        const posZ = 0; 

                        const isGold = Math.random() > 0.3;
                        const mesh = new THREE.Mesh(sphereGeo, isGold ? goldMat : redMat);
                        const s = 0.4 + Math.random() * 0.3;
                        mesh.scale.set(s,s,s);
                        mainGroup.add(mesh);
                        particleSystem.push(new Particle(mesh, 'TEXT', false, new THREE.Vector3(posX, posY, posZ)));
                    }
                }
            }
        }

        function createParticles() {
            const boxGeo = new THREE.BoxGeometry(0.5, 0.5, 0.5); 
            const sphereGeo = new THREE.SphereGeometry(0.5, 32, 32); 
            const greenMat = new THREE.MeshStandardMaterial({ color: CONFIG.colors.deepGreen, metalness: 0.1, roughness: 0.8 });
            
            const curve = new THREE.CatmullRomCurve3([
                new THREE.Vector3(0, -0.5, 0), new THREE.Vector3(0, 0.3, 0),
                new THREE.Vector3(0.1, 0.5, 0), new THREE.Vector3(0.3, 0.4, 0)
            ]);
            const candyGeo = new THREE.TubeGeometry(curve, 16, 0.08, 8, false);
            const candyMat = new THREE.MeshStandardMaterial({ map: caneTexture, roughness: 0.4 });

            for (let i = 0; i < CONFIG.particles.count; i++) {
                const rand = Math.random();
                let mesh, type;
                
                if (rand < 0.6) {
                    mesh = new THREE.Mesh(boxGeo, greenMat);
                    type = 'BOX';
                } else if (rand < 0.8) {
                    mesh = new THREE.Mesh(sphereGeo, goldMat);
                    type = 'GOLD';
                } else if (rand < 0.95) {
                    mesh = new THREE.Mesh(sphereGeo, redMat);
                    type = 'RED';
                } else {
                    mesh = new THREE.Mesh(candyGeo, candyMat);
                    type = 'CANE';
                }

                const s = 0.4 + Math.random() * 0.5;
                mesh.scale.set(s,s,s);
                mesh.rotation.set(Math.random()*6, Math.random()*6, Math.random()*6);
                
                mainGroup.add(mesh);
                particleSystem.push(new Particle(mesh, type, false));
            }

            const starGeo = new THREE.OctahedronGeometry(1.2, 0);
            const starMat = new THREE.MeshStandardMaterial({
                color: 0xffdd88, emissive: 0xffaa00, emissiveIntensity: 1.0,
                metalness: 1.0, roughness: 0
            });
            const star = new THREE.Mesh(starGeo, starMat);
            star.position.set(0, CONFIG.particles.treeHeight/2, 0);
            mainGroup.add(star);
            mainGroup.add(photoMeshGroup);
        }

        function createDust() {
            const geo = new THREE.TetrahedronGeometry(0.08, 0);
            const mat = new THREE.MeshBasicMaterial({ color: 0xffeebb, transparent: true, opacity: 0.6 });
            for(let i=0; i<CONFIG.particles.dustCount; i++) {
                 const mesh = new THREE.Mesh(geo, mat);
                 mesh.scale.setScalar(0.5 + Math.random());
                 mainGroup.add(mesh);
                 particleSystem.push(new Particle(mesh, 'DUST', true));
            }
        }

        function createDefaultPhotos() {
            const canvas = document.createElement('canvas');
            canvas.width = 512; canvas.height = 512;
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = '#050505'; ctx.fillRect(0,0,512,512);
            ctx.strokeStyle = '#eebb66'; ctx.lineWidth = 15; ctx.strokeRect(20,20,472,472);
            ctx.font = '500 80px "SimHei"'; ctx.fillStyle = '#eebb66';
            ctx.textAlign = 'center'; 
            ctx.fillText("平安喜乐", 256, 256);
            
            const tex = new THREE.CanvasTexture(canvas);
            tex.colorSpace = THREE.SRGBColorSpace;
            addPhotoToScene(tex);
        }

        function addPhotoToScene(texture) {
            const frameGeo = new THREE.BoxGeometry(1.4, 1.4, 0.05);
            const frameMat = new THREE.MeshStandardMaterial({ color: CONFIG.colors.champagneGold, metalness: 1.0, roughness: 0.1 });
            const frame = new THREE.Mesh(frameGeo, frameMat);
            const photoGeo = new THREE.PlaneGeometry(1.2, 1.2);
            const photoMat = new THREE.MeshBasicMaterial({ map: texture });
            const photo = new THREE.Mesh(photoGeo, photoMat);
            photo.position.z = 0.04;
            const group = new THREE.Group();
            group.add(frame);
            group.add(photo);
            const s = 1.0;
            group.scale.set(s,s,s);
            photoMeshGroup.add(group);
            particleSystem.push(new Particle(group, 'PHOTO', false));
        }

        async function initMediaPipe() {
            video = document.getElementById('webcam');
            if (navigator.mediaDevices?.getUserMedia) {
                try {
                    const stream = await navigator.mediaDevices.getUserMedia({ 
                        video: { facingMode: "user", width: { ideal: 640 }, height: { ideal: 480 } } 
                    });
                    video.srcObject = stream;
                    video.addEventListener("loadeddata", predictWebcam);
                } catch (e) {
                    console.error("Camera denied", e);
                    alert("请允许摄像头权限以使用手势控制");
                }
            }

            const vision = await FilesetResolver.forVisionTasks(
                "https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@0.10.3/wasm"
            );
            handLandmarker = await HandLandmarker.createFromOptions(vision, {
                baseOptions: {
                    modelAssetPath: `https://storage.googleapis.com/mediapipe-models/hand_landmarker/hand_landmarker/float16/1/hand_landmarker.task`,
                    delegate: "GPU"
                },
                runningMode: "VIDEO",
                numHands: 1
            });
        }

        let lastVideoTime = -1;
        async function predictWebcam() {
            if (video.currentTime !== lastVideoTime) {
                lastVideoTime = video.currentTime;
                if (handLandmarker) {
                    const result = handLandmarker.detectForVideo(video, performance.now());
                    processGestures(result);
                }
            }
            requestAnimationFrame(predictWebcam);
        }

        function processGestures(result) {
            if (result.landmarks && result.landmarks.length > 0) {
                STATE.hand.detected = true;
                const lm = result.landmarks[0];
                STATE.hand.x = (lm[9].x - 0.5) * 2; 
                STATE.hand.y = (lm[9].y - 0.5) * 2;

                const thumb = lm[4]; const index = lm[8]; const wrist = lm[0];
                const pinchDist = Math.hypot(thumb.x - index.x, thumb.y - index.y);
                const tips = [lm[8], lm[12], lm[16], lm[20]];
                let avgDist = 0;
                tips.forEach(t => avgDist += Math.hypot(t.x - wrist.x, t.y - wrist.y));
                avgDist /= 4;

                if (pinchDist < 0.05) {
                    if (STATE.mode !== 'FOCUS') {
                        STATE.mode = 'FOCUS';
                        const photos = particleSystem.filter(p => p.type === 'PHOTO');
                        if (photos.length) STATE.focusTarget = photos[Math.floor(Math.random()*photos.length)].mesh;
                    }
                } else if (avgDist < 0.25) {
                    STATE.mode = 'TREE';
                    STATE.focusTarget = null;
                } else if (avgDist > 0.35) {
                    STATE.mode = 'SCATTER';
                    STATE.focusTarget = null;
                }
            } else {
                STATE.hand.detected = false;
            }
        }

        function setupEvents() {
            window.addEventListener('resize', () => {
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
                composer.setSize(window.innerWidth, window.innerHeight);
            });
            document.getElementById('file-input').addEventListener('change', (e) => {
                const files = e.target.files;
                if(!files.length) return;
                Array.from(files).forEach(f => {
                    const reader = new FileReader();
                    reader.onload = (ev) => {
                        new THREE.TextureLoader().load(ev.target.result, (t) => {
                            t.colorSpace = THREE.SRGBColorSpace;
                            addPhotoToScene(t);
                        });
                    }
                    reader.readAsDataURL(f);
                });
            });
        }

        function animate() {
            requestAnimationFrame(animate);
            const dt = clock.getDelta();

            if (STATE.mode === 'SCATTER' && STATE.hand.detected) {
                const targetRotY = STATE.hand.x * 0.5; 
                const targetRotX = STATE.hand.y * 0.3;
                STATE.rotation.y += (targetRotY - STATE.rotation.y) * 2.0 * dt;
                STATE.rotation.x += (targetRotX - STATE.rotation.x) * 2.0 * dt;
            } else {
                if(STATE.mode === 'TREE') {
                    STATE.rotation.y += 0.3 * dt;
                    STATE.rotation.x += (0 - STATE.rotation.x) * 2.0 * dt;
                } else {
                     STATE.rotation.y += (0 - STATE.rotation.y) * 1.0 * dt;
                     STATE.rotation.x += (0 - STATE.rotation.x) * 1.0 * dt;
                }
            }

            mainGroup.rotation.y = STATE.rotation.y;
            mainGroup.rotation.x = STATE.rotation.x;

            particleSystem.forEach(p => p.update(dt, STATE.mode, STATE.focusTarget));
            composer.render();
        }   

        init();
    </script>
</body>
</html>

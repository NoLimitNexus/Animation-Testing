<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Glowing Orb Simulation</title>
    <style>
        body { margin: 0; overflow: hidden; font-family: 'Inter', sans-serif; background-color: #1a1a2e; color: #fff; }
        canvas { display: block; width: 100%; height: 100vh; }
        #info-box, #controls-box {
            position: absolute;
            background: rgba(0, 0, 0, 0.7);
            padding: 15px;
            border-radius: 10px;
            font-size: 14px;
            box-shadow: 0 0 15px rgba(0, 255, 255, 0.5);
            border: 1px solid #0ff;
            z-index: 100;
        }
        #info-box {
            top: 10px;
            left: 10px;
            max-width: 300px;
        }
        #controls-box {
            top: 10px;
            right: 10px;
            max-width: 300px;
            display: flex;
            flex-direction: column;
            gap: 10px;
        }
        #info-box h3, #controls-box h3 { margin-top: 0; color: #0ff; }
        #info-box p, #controls-box p { margin-bottom: 5px; }
        #controls-box label {
            display: flex;
            justify-content: space-between;
            align-items: center;
            margin-bottom: 5px;
        }
        #controls-box input[type="range"] {
            width: 150px;
            -webkit-appearance: none;
            height: 8px;
            background: #0ff;
            border-radius: 5px;
            outline: none;
            opacity: 0.7;
            transition: opacity .2s;
        }
        #controls-box input[type="range"]::-webkit-slider-thumb {
            -webkit-appearance: none;
            appearance: none;
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: #fff;
            cursor: pointer;
            box-shadow: 0 0 5px rgba(0, 255, 255, 0.8);
        }
        #controls-box input[type="range"]::-moz-range-thumb {
            width: 20px;
            height: 20px;
            border-radius: 50%;
            background: #fff;
            cursor: pointer;
            box-shadow: 0 0 5px rgba(0, 255, 255, 0.8);
        }
        #controls-box button {
            background-color: #0ff;
            color: #1a1a2e;
            border: none;
            padding: 10px 15px;
            border-radius: 5px;
            cursor: pointer;
            font-weight: bold;
            transition: background-color 0.3s ease, transform 0.1s ease;
            box-shadow: 0 0 10px rgba(0, 255, 255, 0.5);
        }
        #controls-box button:hover {
            background-color: #00cccc;
            transform: translateY(-2px);
        }
        #controls-box button:active {
            transform: translateY(0);
        }
    </style>
    <!-- Three.js CDN -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <!-- OrbitControls for camera movement -->
    <script src="https://cdn.jsdelivr.net/npm/three@0.128.0/examples/js/controls/OrbitControls.js"></script>
    <!-- Cannon.js CDN for physics -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/cannon.js/0.6.2/cannon.min.js"></script>
</head>
<body>
    <div id="info-box">
        <h3>Simulation Info</h3>
        <p><strong>Mouse Left Click:</strong> Rotate Camera</p>
        <p><strong>Mouse Right Click:</strong> Pan Camera</p>
        <p><strong>Mouse Scroll:</strong> Zoom In/Out</p>
        <p><strong>Goal:</strong> Observe the glowing orbs collect materials and return to the central base.</p>
        <p><strong>Orbs:</strong> <span id="orb-count">0</span></p>
        <p><strong>Materials:</strong> <span id="material-count">0</span></p>
        <p><strong>Collected:</strong> <span id="collected-count">0</span></p>
    </div>

    <div id="controls-box">
        <h3>Simulation Controls</h3>
        <label>
            Orb Speed: <span id="orb-speed-value">5</span>
            <input type="range" id="orb-speed-slider" min="1" max="20" value="5" step="0.5">
        </label>
        <label>
            Materials Per Orb: <span id="materials-per-orb-value">1</span>
            <input type="range" id="materials-per-orb-slider" min="1" max="5" value="1" step="1">
        </label>
        <label>
            Map Size: <span id="map-size-value">100</span>
            <input type="range" id="map-size-slider" min="50" max="200" value="100" step="10">
        </label>
        <label>
            Initial Resources: <span id="initial-resources-value">50</span>
            <input type="range" id="initial-resources-slider" min="10" max="200" value="50" step="10">
        </label>
        <button id="reset-simulation-button">Reset Simulation</button>
    </div>

    <script>
        // --- Global Variables ---
        let scene, camera, renderer, controls;
        let world; // Cannon.js physics world

        const orbs = [];
        const materials = [];
        let baseMesh, baseBody;
        let groundMesh, groundBody; // Make ground variables global for dynamic updates
        let directionalLight; // Make directional light global for dynamic updates

        // Simulation parameters, now linked to sliders
        let currentOrbSpeed = 5;
        let maxCarriedMaterialsPerOrb = 1;
        let currentGroundSize = 100;
        let currentMaterialCount = 50;

        const ORB_COUNT = 10; // Fixed number of orbs
        const ORB_RADIUS = 0.8;
        const MATERIAL_SIZE = 0.5;
        const BASE_RADIUS = 5;

        let collectedMaterialsCount = 0;
        let animationFrameId; // To keep track of the animation frame

        // --- Utility Functions ---

        // Generates a random position within the current ground bounds
        function getRandomGroundPosition() {
            return new THREE.Vector3(
                (Math.random() - 0.5) * currentGroundSize * 0.8,
                MATERIAL_SIZE / 2, // Slightly above ground
                (Math.random() - 0.5) * currentGroundSize * 0.8
            );
        }

        // --- Initialization ---

        function init(groundSize, materialCount) {
            // Cancel any ongoing animation frame before re-initializing
            if (animationFrameId) {
                cancelAnimationFrame(animationFrameId);
                animationFrameId = null; // Clear the ID
            }

            // Dispose of existing Three.js objects and remove renderer from DOM
            if (renderer) {
                if (controls) {
                    controls.dispose(); // Dispose OrbitControls first
                    controls = null; // Clear the reference
                }
                if (renderer.domElement.parentNode) {
                    renderer.domElement.parentNode.removeChild(renderer.domElement);
                }
                renderer.dispose(); // Dispose WebGLRenderer
                renderer = null; // Clear the reference
            }

            // Clear scene children and dispose of their geometries/materials
            if (scene) {
                // Iterate over a copy of the children array to avoid issues during removal
                scene.children.slice().forEach(child => {
                    // Recursively dispose children if they have dispose methods
                    if (child.dispose) child.dispose(); // For lights, cameras, etc.
                    if (child.geometry) child.geometry.dispose();
                    if (child.material) {
                        if (Array.isArray(child.material)) {
                            child.material.forEach(mat => mat.dispose());
                        } else {
                            child.material.dispose();
                        }
                    }
                    // Remove from scene
                    scene.remove(child);
                });
                scene = null; // Clear the reference
            }

            // Clear arrays holding references to old objects
            orbs.length = 0;
            materials.length = 0;

            // Re-initialize world or clear existing one
            if (world) {
                // Remove all bodies from the old world
                while(world.bodies.length > 0) {
                    world.removeBody(world.bodies[0]);
                }
            } else {
                world = new CANNON.World();
            }

            // Update current parameters for this initialization
            currentGroundSize = groundSize;
            currentMaterialCount = materialCount;
            collectedMaterialsCount = 0; // Reset collected count on simulation reset

            // Scene setup
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x1a1a2e); // Dark background

            // Camera setup
            camera = new THREE.PerspectiveCamera(75, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.set(0, 50, currentGroundSize * 0.7); // Adjust camera based on ground size

            // Renderer setup
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true; // Enable shadow maps
            renderer.shadowMap.type = THREE.PCFSoftShadowMap; // Soft shadows
            document.body.appendChild(renderer.domElement); // Append new renderer DOM element

            // OrbitControls for camera interaction (must be initialized AFTER renderer.domElement is in DOM)
            controls = new THREE.OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true; // Enable smooth camera movement
            controls.dampingFactor = 0.05;
            controls.screenSpacePanning = false;
            controls.maxPolarAngle = Math.PI / 2 - 0.1; // Prevent camera from going below ground

            // Lighting
            const hemisphereLight = new THREE.HemisphereLight(0xffffff, 0x444444, 0.6);
            scene.add(hemisphereLight);

            directionalLight = new THREE.DirectionalLight(0xffffff, 0.8); // Assign to global variable
            directionalLight.position.set(20, 50, 20);
            directionalLight.castShadow = true;
            directionalLight.shadow.mapSize.width = 2048;
            directionalLight.shadow.mapSize.height = 2048;
            directionalLight.shadow.camera.near = 0.5;
            directionalLight.shadow.camera.far = 150;
            // Adjust shadow camera bounds based on ground size
            directionalLight.shadow.camera.left = -currentGroundSize / 2;
            directionalLight.shadow.camera.right = currentGroundSize / 2;
            directionalLight.shadow.camera.top = currentGroundSize / 2;
            directionalLight.shadow.camera.bottom = -currentGroundSize / 2;
            scene.add(directionalLight);

            // Cannon.js physics world setup
            world.gravity.set(0, -9.82, 0); // Standard gravity
            world.broadphase = new CANNON.SAPBroadphase(world); // Optimized broadphase
            world.solver.iterations = 10; // Increase iterations for stability

            // Create Ground
            createGround();

            // Create Base
            createBase();

            // Create Orbs
            for (let i = 0; i < ORB_COUNT; i++) {
                createOrb();
            }

            // Create Materials
            for (let i = 0; i < currentMaterialCount; i++) {
                createMaterial();
            }

            // Event Listeners for window resize (always active)
            window.removeEventListener('resize', onWindowResize); // Remove previous listener if exists
            window.addEventListener('resize', onWindowResize);

            // Update UI counts
            updateInfoBox();
        }

        // --- Object Creation Functions ---

        function createGround() {
            // Dispose of old ground if it exists
            if (groundMesh) {
                scene.remove(groundMesh);
                groundMesh.geometry.dispose();
                groundMesh.material.dispose();
                groundMesh = null;
            }
            if (groundBody) {
                world.removeBody(groundBody);
                groundBody = null;
            }

            // Three.js ground mesh
            const groundGeometry = new THREE.PlaneGeometry(currentGroundSize, currentGroundSize);
            const groundMaterial = new THREE.MeshStandardMaterial({
                color: 0x333333,
                roughness: 0.8,
                metalness: 0.1,
                side: THREE.DoubleSide
            });
            groundMesh = new THREE.Mesh(groundGeometry, groundMaterial);
            groundMesh.rotation.x = -Math.PI / 2;
            groundMesh.receiveShadow = true; // Ground receives shadows
            scene.add(groundMesh);

            // Cannon.js ground body
            const groundShape = new CANNON.Plane();
            groundBody = new CANNON.Body({ mass: 0 }); // Mass 0 makes it static
            groundBody.addShape(groundShape);
            groundBody.quaternion.setFromAxisAngle(new CANNON.Vec3(1, 0, 0), -Math.PI / 2); // Rotate to align with Three.js ground
            world.addBody(groundBody);
        }

        function createBase() {
            // Three.js base mesh
            const baseGeometry = new THREE.CylinderGeometry(BASE_RADIUS, BASE_RADIUS, 2, 32);
            const baseMaterial = new THREE.MeshStandardMaterial({
                color: 0x00ffff, // Cyan glow
                emissive: 0x00ffff,
                emissiveIntensity: 0.5,
                roughness: 0.5,
                metalness: 0.2
            });
            baseMesh = new THREE.Mesh(baseGeometry, baseMaterial);
            baseMesh.position.set(0, 1, 0);
            baseMesh.castShadow = true;
            baseMesh.receiveShadow = true;
            scene.add(baseMesh);

            // Cannon.js base body (static)
            const baseShape = new CANNON.Cylinder(BASE_RADIUS, BASE_RADIUS, 2, 32);
            baseBody = new CANNON.Body({ mass: 0 });
            baseBody.addShape(baseShape);
            baseBody.position.copy(baseMesh.position);
            // Cannon.js cylinder is aligned with Y-axis by default, so no rotation needed if mesh is also aligned.
            world.addBody(baseBody);
        }

        function createOrb() {
            // Three.js orb mesh
            const orbGeometry = new THREE.SphereGeometry(ORB_RADIUS, 32, 32);
            const orbMaterial = new THREE.MeshStandardMaterial({
                color: 0xffa500, // Orange glow
                emissive: 0xffa500,
                emissiveIntensity: 1.0,
                roughness: 0.5,
                metalness: 0.2
            });
            const orbMesh = new THREE.Mesh(orbGeometry, orbMaterial);
            orbMesh.position.copy(getRandomGroundPosition());
            orbMesh.castShadow = true;
            scene.add(orbMesh);

            // Add a point light to the orb for a stronger glow effect
            const orbLight = new THREE.PointLight(0xffa500, 1.5, 10); // Color, intensity, distance
            orbLight.position.set(0, 0, 0); // Relative to orb mesh
            orbMesh.add(orbLight); // Add light as child of orb mesh

            // Cannon.js orb body
            const orbShape = new CANNON.Sphere(ORB_RADIUS);
            const orbBody = new CANNON.Body({
                mass: 1, // Orbs have mass
                shape: orbShape,
                position: new CANNON.Vec3(orbMesh.position.x, orbMesh.position.y, orbMesh.position.z),
                linearDamping: 0.9, // Reduce bouncing
                angularDamping: 0.9
            });
            world.addBody(orbBody);

            // Orb state
            const orb = {
                mesh: orbMesh,
                body: orbBody,
                state: 'seeking', // 'seeking', 'carrying'
                targetMaterial: null, // Not used directly for targeting anymore, but kept for reference
                carriedMaterials: [], // Now an array to hold multiple materials
                id: orbs.length // Unique ID for collision detection
            };
            orbs.push(orb);

            // Collision listener for orbs
            orbBody.addEventListener('collide', (event) => {
                const otherBody = event.body;

                // Check for collision with materials
                const collidedMaterial = materials.find(m => m.body === otherBody && m.active);
                if (collidedMaterial && orb.state === 'seeking' && orb.carriedMaterials.length < maxCarriedMaterialsPerOrb) {
                    orb.carriedMaterials.push(collidedMaterial); // Add to carried materials
                    collidedMaterial.active = false; // Deactivate material on ground
                    world.removeBody(collidedMaterial.body); // Remove its physics body

                    // Attach material mesh to orb mesh, stacking them
                    orb.mesh.add(collidedMaterial.mesh);
                    const stackHeight = ORB_RADIUS + (orb.carriedMaterials.length - 1) * MATERIAL_SIZE + MATERIAL_SIZE / 2;
                    collidedMaterial.mesh.position.set(0, stackHeight, 0);
                    collidedMaterial.mesh.rotation.set(0, 0, 0); // Reset its relative rotation

                    orb.state = 'carrying'; // Change state to carrying once it picks up the first material
                    updateInfoBox();
                }

                // Check for collision with base
                if (otherBody === baseBody && orb.state === 'carrying' && orb.carriedMaterials.length > 0) {
                    // Drop all carried materials
                    orb.carriedMaterials.forEach(carriedMat => {
                        // Detach material from orb mesh
                        orb.mesh.remove(carriedMat.mesh);

                        collectedMaterialsCount++; // Increment collected count for each material

                        // Immediately respawn the material at a new random location
                        carriedMat.active = true; // Make it active again
                        carriedMat.mesh.position.copy(getRandomGroundPosition()); // Set new world position
                        carriedMat.body.position.copy(carriedMat.mesh.position); // Sync physics body
                        scene.add(carriedMat.mesh); // Add back to main scene
                        world.addBody(carriedMat.body); // Add physics body back to world
                    });

                    orb.carriedMaterials.length = 0; // Clear the array
                    orb.state = 'seeking'; // Orb goes back to seeking
                    updateInfoBox();
                }
            });
            updateInfoBox();
        }

        function createMaterial() {
            // Three.js material mesh
            const materialGeometry = new THREE.BoxGeometry(MATERIAL_SIZE, MATERIAL_SIZE, MATERIAL_SIZE);
            const materialMaterial = new THREE.MeshStandardMaterial({
                color: 0x00ff00, // Green material
                emissive: 0x00ff00,
                emissiveIntensity: 0.7,
                roughness: 0.5,
                metalness: 0.2
            });
            const materialMesh = new THREE.Mesh(materialGeometry, materialMaterial);
            materialMesh.position.copy(getRandomGroundPosition());
            materialMesh.castShadow = true;
            scene.add(materialMesh);

            // Cannon.js material body (static for simplicity, or dynamic if orbs push them)
            const materialShape = new CANNON.Box(new CANNON.Vec3(MATERIAL_SIZE / 2, MATERIAL_SIZE / 2, MATERIAL_SIZE / 2));
            const materialBody = new CANNON.Body({
                mass: 0.1, // Small mass so orbs can push them slightly
                shape: materialShape,
                position: new CANNON.Vec3(materialMesh.position.x, materialMesh.position.y, materialMesh.position.z)
            });
            world.addBody(materialBody);

            const material = {
                mesh: materialMesh,
                body: materialBody,
                active: true,
            };
            materials.push(material);
            updateInfoBox();
        }

        // --- Animation Loop ---

        const timeStep = 1 / 60; // seconds

        function animate() {
            animationFrameId = requestAnimationFrame(animate); // Update animationFrameId here

            // Update physics world
            world.step(timeStep);

            // Update orbs
            orbs.forEach(orb => {
                // Sync Three.js mesh with Cannon.js body
                orb.mesh.position.copy(orb.body.position);
                orb.mesh.quaternion.copy(orb.body.quaternion);

                let targetPosition;

                if (orb.state === 'seeking') {
                    // Find the nearest active material
                    let nearestMaterial = null;
                    let minDistance = Infinity;

                    materials.forEach(material => {
                        if (material.active) {
                            const distance = orb.mesh.position.distanceTo(material.mesh.position);
                            if (distance < minDistance) {
                                minDistance = distance;
                                nearestMaterial = material;
                            }
                        }
                    });

                    if (nearestMaterial) {
                        targetPosition = nearestMaterial.mesh.position;
                    } else {
                        // If no materials, just wander
                        targetPosition = getRandomGroundPosition(); // Wander
                    }
                } else if (orb.state === 'carrying') {
                    targetPosition = baseMesh.position;
                }

                if (targetPosition) {
                    const direction = new THREE.Vector3().subVectors(targetPosition, orb.mesh.position).normalize();
                    // Apply force to the orb's physics body
                    // Ensure the orb stays on the ground plane for movement
                    const force = new CANNON.Vec3(direction.x, 0, direction.z).scale(currentOrbSpeed * orb.body.mass);
                    orb.body.applyForce(force, orb.body.position);

                    // Limit orb speed to prevent it from flying off
                    const maxSpeed = 10;
                    if (orb.body.velocity.length() > maxSpeed) {
                        orb.body.velocity.normalize();
                        orb.body.velocity.scale(maxSpeed);
                    }
                }
            });

            // Update controls and render scene
            // Only update controls if they are not null
            if (controls) {
                controls.update();
            }
            // Only render if renderer is not null
            if (renderer) {
                renderer.render(scene, camera);
            }
        }

        // --- Event Handlers ---

        function onWindowResize() {
            if (camera && renderer) { // Ensure camera and renderer exist before updating
                camera.aspect = window.innerWidth / window.innerHeight;
                camera.updateProjectionMatrix();
                renderer.setSize(window.innerWidth, window.innerHeight);
            }
        }

        function updateInfoBox() {
            document.getElementById('orb-count').textContent = orbs.length;
            document.getElementById('material-count').textContent = materials.filter(m => m.active).length;
            document.getElementById('collected-count').textContent = collectedMaterialsCount;
        }

        // --- Slider and Button Logic ---
        window.onload = function () {
            // Initial call with default values
            init(currentGroundSize, currentMaterialCount);
            animate(); // Start the animation loop

            const orbSpeedSlider = document.getElementById('orb-speed-slider');
            const orbSpeedValue = document.getElementById('orb-speed-value');
            const materialsPerOrbSlider = document.getElementById('materials-per-orb-slider');
            const materialsPerOrbValue = document.getElementById('materials-per-orb-value');
            const mapSizeSlider = document.getElementById('map-size-slider');
            const mapSizeValue = document.getElementById('map-size-value');
            const initialResourcesSlider = document.getElementById('initial-resources-slider');
            const initialResourcesValue = document.getElementById('initial-resources-value');
            const resetButton = document.getElementById('reset-simulation-button');

            // Initialize slider values
            orbSpeedSlider.value = currentOrbSpeed;
            orbSpeedValue.textContent = currentOrbSpeed;
            materialsPerOrbSlider.value = maxCarriedMaterialsPerOrb;
            materialsPerOrbValue.textContent = maxCarriedMaterialsPerOrb;
            mapSizeSlider.value = currentGroundSize;
            mapSizeValue.textContent = currentGroundSize;
            initialResourcesSlider.value = currentMaterialCount;
            initialResourcesValue.textContent = currentMaterialCount;


            orbSpeedSlider.addEventListener('input', (event) => {
                currentOrbSpeed = parseFloat(event.target.value);
                orbSpeedValue.textContent = currentOrbSpeed;
            });

            materialsPerOrbSlider.addEventListener('input', (event) => {
                maxCarriedMaterialsPerOrb = parseInt(event.target.value);
                materialsPerOrbValue.textContent = maxCarriedMaterialsPerOrb;
            });

            mapSizeSlider.addEventListener('input', (event) => {
                const newSize = parseInt(event.target.value);
                if (newSize !== currentGroundSize) {
                    currentGroundSize = newSize;
                    mapSizeValue.textContent = currentGroundSize;
                    // Dynamically update ground and camera
                    createGround(); // Recreates and adds ground to scene/world
                    camera.position.set(0, 50, currentGroundSize * 0.7); // Adjust camera based on new ground size
                    directionalLight.shadow.camera.left = -currentGroundSize / 2;
                    directionalLight.shadow.camera.right = currentGroundSize / 2;
                    directionalLight.shadow.camera.top = currentGroundSize / 2;
                    directionalLight.shadow.camera.bottom = -currentGroundSize / 2;
                    directionalLight.shadow.camera.updateProjectionMatrix();
                    controls.update(); // Update controls to reflect new camera position
                }
            });

            initialResourcesSlider.addEventListener('input', (event) => {
                const newMaterialCount = parseInt(event.target.value);
                if (newMaterialCount !== currentMaterialCount) {
                    const oldMaterialCount = currentMaterialCount;
                    currentMaterialCount = newMaterialCount;
                    initialResourcesValue.textContent = currentMaterialCount;

                    const diff = newMaterialCount - oldMaterialCount;

                    if (diff > 0) {
                        // Add new materials
                        for (let i = 0; i < diff; i++) {
                            createMaterial();
                        }
                    } else if (diff < 0) {
                        // Remove materials
                        const materialsToRemoveCount = Math.abs(diff);
                        let removedCount = 0;

                        // Prioritize removing inactive materials first
                        for (let i = materials.length - 1; i >= 0 && removedCount < materialsToRemoveCount; i--) {
                            if (!materials[i].active) {
                                const material = materials.splice(i, 1)[0]; // Remove from array
                                scene.remove(material.mesh);
                                // No need to remove body from world if it's already inactive (removed earlier)
                                removedCount++;
                            }
                        }

                        // If still more to remove, remove active ones
                        for (let i = materials.length - 1; i >= 0 && removedCount < materialsToRemoveCount; i--) {
                            const material = materials.splice(i, 1)[0]; // Remove from array
                            scene.remove(material.mesh);
                            world.removeBody(material.body); // Remove its physics body
                            removedCount++;
                        }
                    }
                    updateInfoBox();
                }
            });

            resetButton.addEventListener('click', () => {
                init(currentGroundSize, currentMaterialCount);
                // The animate() call within init will restart the loop
            });
        };
    </script>
</body>
</html>

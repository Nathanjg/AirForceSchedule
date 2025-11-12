
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Flight Deconfliction Schedule (Firestore)</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <style>
        /* Custom styles for a cleaner look */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700&display=swap');
        body {
            font-family: 'Inter', sans-serif;
            background-color: #f7f7f9;
        }
        .scrollable-table {
            max-height: 70vh;
            overflow-y: auto;
        }
    </style>
</head>
<body class="p-4 sm:p-8">
    <div class="max-w-6xl mx-auto bg-white shadow-xl rounded-xl p-6 lg:p-10">
        <h1 class="text-3xl font-extrabold text-gray-900 mb-6 border-b pb-2">Airspace Schedule Manager (Live Data)</h1>
        <p class="text-gray-600 mb-4">Flagging potential conflicts at 8 interception points. Minimum separation time is **5 minutes**.</p>
        <p id="userIdDisplay" class="text-sm text-gray-500 mb-8"></p>

        <!-- Flight Input Form -->
        <div class="mb-10 p-6 bg-blue-50 rounded-lg border border-blue-200">
            <h2 class="text-xl font-semibold text-blue-800 mb-4">Add New Flight</h2>
            <form id="addFlightForm" class="space-y-4">
                <!-- Basic Flight Info -->
                <div class="grid grid-cols-1 md:grid-cols-3 gap-4">
                    <input type="text" id="flightId" required placeholder="Flight ID (e.g., UA123)" class="p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500">
                    <input type="text" id="origin" required placeholder="Origin" class="p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500">
                    <input type="text" id="destination" required placeholder="Destination" class="p-3 border border-gray-300 rounded-lg focus:ring-blue-500 focus:border-blue-500">
                </div>

                <!-- Intersection Times (8 Spots) -->
                <h3 class="text-lg font-medium text-gray-700 mt-6 pt-4 border-t">8 Intersection Timings (Local Date/Time)</h3>
                <p class="text-sm text-gray-500 mb-2">Use the format `YYYY-MM-DDTHH:MM` (e.g., 2025-12-31T15:30) for each spot.</p>
                <div id="intersectionInputs" class="grid grid-cols-2 sm:grid-cols-4 gap-4">
                    <!-- Inputs will be generated here by JavaScript -->
                </div>
                
                <button type="submit" id="submitButton" class="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-lg transition duration-150 shadow-md">
                    Add Flight & Check Deconfliction
                </button>
            </form>
        </div>
        
        <!-- Flight Schedule Table -->
        <h2 class="text-xl font-semibold text-gray-900 mb-4">Current Flight Schedule</h2>
        <div class="scrollable-table shadow-lg rounded-lg border border-gray-200">
            <table class="min-w-full divide-y divide-gray-300">
                <thead class="sticky top-0 bg-gray-100 z-10">
                    <tr>
                        <th class="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Flight ID</th>
                        <th class="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider hidden sm:table-cell">Route</th>
                        <th class="px-3 py-3 text-center text-xs font-medium text-gray-500 uppercase tracking-wider">Status</th>
                        <th class="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider hidden md:table-cell">Conflict Spot(s)</th>
                        <th class="px-3 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">Actions</th>
                    </tr>
                </thead>
                <tbody id="scheduleBody" class="bg-white divide-y divide-gray-200">
                    <!-- Flight rows will be inserted here -->
                </tbody>
            </table>
        </div>
        <div id="messageBox" class="mt-4 p-4 rounded-lg hidden" role="alert"></div>

    </div>

    <!-- Firebase Imports -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, deleteDoc, onSnapshot, collection, query, addDoc, setLogLevel, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- Configuration ---
        const NUM_INTERSECTIONS = 8;
        // Minimum separation time in milliseconds (5 minutes * 60 seconds/min * 1000 ms/sec)
        const MIN_SEPARATION_MS = 5 * 60 * 1000;
        
        // --- Firebase Globals ---
        let db;
        let auth;
        let userId = 'loading...';
        let appId;
        
        // Local state array (populated by Firestore listener)
        let flights = [];

        // --- DOM Elements ---
        const form = document.getElementById('addFlightForm');
        const scheduleBody = document.getElementById('scheduleBody');
        const intersectionInputsContainer = document.getElementById('intersectionInputs');
        const messageBox = document.getElementById('messageBox');
        const userIdDisplay = document.getElementById('userIdDisplay');
        const submitButton = document.getElementById('submitButton');


        /**
         * Converts date/time string to milliseconds for comparison.
         * @param {string} dateTimeString - ISO format date/time string (e.g., 2025-11-15T10:15).
         * @returns {number} Milliseconds since epoch, or 0 if invalid.
         */
        function getTimeInMs(dateTimeString) {
            const time = new Date(dateTimeString).getTime();
            return isNaN(time) ? 0 : time;
        }

        /**
         * Displays a temporary message to the user (e.g., success or error).
         * @param {string} message - The message content.
         * @param {string} type - 'success' or 'error'.
         */
        function showMessage(message, type = 'success') {
            messageBox.textContent = message;
            messageBox.classList.remove('hidden', 'bg-green-100', 'text-green-800', 'bg-red-100', 'text-red-800');

            if (type === 'success') {
                messageBox.classList.add('bg-green-100', 'text-green-800');
            } else if (type === 'error') {
                messageBox.classList.add('bg-red-100', 'text-red-800');
            }

            setTimeout(() => {
                messageBox.classList.add('hidden');
            }, 5000);
        }

        /**
         * Core logic to check for conflicts against all flights in the provided list.
         * This function calculates the *live* conflict status based on the current data.
         * @param {Array<Object>} flightsList - The complete list of flights from the database (immutable data).
         * @returns {Array<Object>} - A new array of flights with live 'status' and 'conflictSpots' added.
         */
        function getLiveSchedule(flightsList) {
            // Create a working copy to add transient conflict status fields
            const liveSchedule = flightsList.map(f => ({ 
                ...f, 
                status: 'OK', 
                conflictSpots: [] 
            }));

            for (let i = 0; i < liveSchedule.length; i++) {
                for (let j = i + 1; j < liveSchedule.length; j++) {
                    const flightA = liveSchedule[i];
                    const flightB = liveSchedule[j];

                    for (let spotIndex = 0; spotIndex < NUM_INTERSECTIONS; spotIndex++) {
                        const timeA = getTimeInMs(flightA.intersectionTimes[spotIndex]);
                        const timeB = getTimeInMs(flightB.intersectionTimes[spotIndex]);

                        // Check for time difference
                        if (Math.abs(timeA - timeB) < MIN_SEPARATION_MS && timeA !== 0 && timeB !== 0) {
                            // Conflict detected at this spot!
                            const spotNumber = spotIndex + 1;

                            // Update flight A status
                            flightA.status = 'CONFLICT';
                            if (!flightA.conflictSpots.includes(spotNumber)) {
                                flightA.conflictSpots.push(spotNumber);
                            }

                            // Update flight B status
                            flightB.status = 'CONFLICT';
                            if (!flightB.conflictSpots.includes(spotNumber)) {
                                flightB.conflictSpots.push(spotNumber);
                            }
                        }
                    }
                }
            }
            return liveSchedule;
        }

        /**
         * Renders the flight table based on the current 'flights' array (which includes live status).
         */
        function renderSchedule() {
            scheduleBody.innerHTML = ''; // Clear existing rows

            if (flights.length === 0) {
                scheduleBody.innerHTML = `
                    <tr>
                        <td colspan="5" class="px-3 py-4 text-center text-gray-500 italic">No flights scheduled. Add one above!</td>
                    </tr>
                `;
                return;
            }

            flights.forEach(flight => {
                const isConflict = flight.status === 'CONFLICT';
                const rowClass = isConflict ? 'bg-red-100 hover:bg-red-200' : 'bg-white hover:bg-gray-50';
                const statusBadgeClass = isConflict 
                    ? 'bg-red-500 text-white' 
                    : 'bg-green-100 text-green-800';

                const conflictText = isConflict 
                    ? `Spot(s): ${flight.conflictSpots.sort((a, b) => a - b).join(', ')}`
                    : '—';

                const row = document.createElement('tr');
                row.className = rowClass + ' transition-colors';
                row.innerHTML = `
                    <td class="px-3 py-3 whitespace-nowrap text-sm font-medium text-gray-900">${flight.flightId}</td>
                    <td class="px-3 py-3 whitespace-nowrap text-sm text-gray-500 hidden sm:table-cell">${flight.origin} → ${flight.destination}</td>
                    <td class="px-3 py-3 whitespace-nowrap text-center">
                        <span class="px-2 inline-flex text-xs leading-5 font-semibold rounded-full ${statusBadgeClass}">
                            ${flight.status}
                        </span>
                    </td>
                    <td class="px-3 py-3 whitespace-nowrap text-sm text-gray-700 hidden md:table-cell">${conflictText}</td>
                    <td class="px-3 py-3 whitespace-nowrap text-right text-sm font-medium">
                        <button onclick="deleteFlight('${flight.id}')" class="text-red-600 hover:text-red-900 text-xs font-semibold p-1 rounded-md hover:bg-red-100 transition">Delete</button>
                    </td>
                `;
                scheduleBody.appendChild(row);
            });
        }
        
        /**
         * Generates the 8 required time inputs for intersection times.
         */
        function generateIntersectionInputs() {
            let html = '';
            for (let i = 1; i <= NUM_INTERSECTIONS; i++) {
                html += `
                    <div>
                        <label for="intTime${i}" class="block text-xs font-medium text-gray-700 mb-1">Spot ${i}</label>
                        <input type="datetime-local" id="intTime${i}" required 
                               placeholder="Spot ${i} Time" 
                               class="p-3 border border-gray-300 rounded-lg w-full text-sm focus:ring-blue-500 focus:border-blue-500">
                    </div>
                `;
            }
            intersectionInputsContainer.innerHTML = html;
        }

        /**
         * Handles form submission to add a new flight.
         */
        form.addEventListener('submit', async function(e) {
            e.preventDefault();
            
            if (!db) {
                showMessage("Database is not initialized. Please wait or check initialization.", 'error');
                return;
            }

            submitButton.disabled = true;
            submitButton.textContent = 'Adding Flight...';

            const flightId = document.getElementById('flightId').value.trim().toUpperCase();
            const origin = document.getElementById('origin').value.trim().toUpperCase();
            const destination = document.getElementById('destination').value.trim().toUpperCase();

            // Collect all 8 intersection times
            const intersectionTimes = [];
            let allTimesValid = true;
            for (let i = 1; i <= NUM_INTERSECTIONS; i++) {
                const timeValue = document.getElementById(`intTime${i}`).value;
                if (!timeValue) {
                    allTimesValid = false;
                    break;
                }
                intersectionTimes.push(timeValue);
            }

            if (!allTimesValid) {
                showMessage("Please fill in all 8 intersection times.", 'error');
                submitButton.disabled = false;
                submitButton.textContent = 'Add Flight & Check Deconfliction';
                return;
            }

            const newFlightData = {
                flightId,
                origin,
                destination,
                intersectionTimes,
                createdAt: serverTimestamp(),
                createdBy: userId
            };

            // Save to Firestore. The onSnapshot listener will handle conflict checking and re-rendering.
            try {
                // The collection path uses the Project ID as the App ID for data separation
                const collectionPath = `artifacts/${appId}/public/data/flights`;
                await addDoc(collection(db, collectionPath), newFlightData);
                
                showMessage(`Flight ${flightId} added successfully. Schedule will update in real-time.`, 'success');

            } catch (error) {
                console.error("Error saving flight:", error);
                showMessage(`Error saving flight: ${error.message}`, 'error');
            } finally {
                // Reset form inputs and button state
                form.reset();
                submitButton.disabled = false;
                submitButton.textContent = 'Add Flight & Check Deconfliction';
            }
        });

        /**
         * Deletes a flight by ID from Firestore.
         * @param {string} id - The document ID of the flight to delete.
         */
        window.deleteFlight = async function(id) {
            try {
                const docPath = `artifacts/${appId}/public/data/flights/${id}`;
                await deleteDoc(doc(db, docPath));
                showMessage('Flight deletion requested. Schedule re-evaluated.', 'success');
            } catch (error) {
                console.error("Error deleting flight:", error);
                showMessage(`Error deleting flight: ${error.message}`, 'error');
            }
        }
        
        /**
         * Sets up the real-time listener to Firestore.
         */
        function setupRealtimeListener(currentAppId) {
            const flightsColRef = collection(db, `artifacts/${currentAppId}/public/data/flights`);
            const q = query(flightsColRef);

            onSnapshot(q, (snapshot) => {
                const flightsFromDb = [];
                snapshot.forEach(doc => {
                    flightsFromDb.push({
                        id: doc.id,
                        ...doc.data()
                    });
                });

                // Calculate the live conflict status locally
                flights = getLiveSchedule(flightsFromDb);
                
                // Re-render the schedule table
                renderSchedule();
            }, (error) => {
                console.error("Firestore snapshot error:", error);
                if (db) {
                    showMessage(`Data retrieval failed: ${error.message}. Check your security rules!`, 'error');
                }
            });
        }


        /**
         * Initializes Firebase and authenticates the user.
         */
        async function initializeFirebase() {
            
            // --- STEP 1: YOUR LIVE FIREBASE CONFIGURATION ---
            // This object contains the keys you provided.
            const firebaseConfig = {
              apiKey: "AIzaSyAXeuJ7JX5Wk-nJJcmuiJUyka7VhFOfFBM",
              authDomain: "schedulebackend-3f971.firebaseapp.com",
              projectId: "schedulebackend-3f971",
              storageBucket: "schedulebackend-3f971.firebasestorage.app",
              messagingSenderId: "515254394901",
              appId: "1:515254394901:web:2d1d175efa23b9849a6e1c",
              measurementId: "G-72W7XFDTZT"
            };
            
            // We use the projectId as the unique application ID for data partitioning in Firestore.
            appId = firebaseConfig.projectId; 
            
            try {
                const app = initializeApp(firebaseConfig);
                db = getFirestore(app);
                auth = getAuth(app);
                setLogLevel('debug'); // Enable detailed Firebase logging

                // 1. Authentication (Using Anonymous sign-in for simplicity)
                await signInAnonymously(auth);
                
                // Get the User ID after sign-in
                userId = auth.currentUser?.uid || crypto.randomUUID();
                userIdDisplay.textContent = `Authenticated User ID: ${userId}`;
                
                // 2. Start listening for data
                setupRealtimeListener(appId);

            } catch (error) {
                console.error("Firebase Initialization/Auth failed:", error);
                showMessage('Initialization failed. Check your config and services in the Firebase console.', 'error');
                userIdDisplay.textContent = `User ID: Error`;
            }
        }

        // --- Initialization ---
        document.addEventListener('DOMContentLoaded', () => {
            generateIntersectionInputs();
            initializeFirebase();
        });

    </script>
</body>
</html>

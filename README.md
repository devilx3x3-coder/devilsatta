<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Coin Hunt Pro - Play & Earn</title>
    
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/9.6.1/firebase-database-compat.js"></script>

    <style>
        :root { --gold: #ffcc00; --blue: #00d4ff; --dark: #121212; --card: #1e1e1e; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--dark); color: white; margin: 0; padding: 20px; text-align: center; }
        .grid-container { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; max-width: 900px; margin: auto; }
        .card { background: var(--card); padding: 15px; border-radius: 12px; border: 1px solid #333; margin-bottom: 15px; }
        
        canvas { background: #000; border: 2px solid var(--gold); border-radius: 10px; width: 100%; max-width: 500px; cursor: pointer; }
        
        button { padding: 10px 20px; border-radius: 8px; border: none; font-weight: bold; cursor: pointer; transition: 0.2s; margin: 5px; }
        .btn-pay { background: var(--blue); color: black; width: 100%; font-size: 16px; }
        .btn-submit { background: var(--gold); color: black; }
        .btn-admin { background: #444; color: white; font-size: 10px; margin-top: 50px; }
        
        .mailbox { text-align: left; height: 120px; overflow-y: auto; background: #151515; padding: 10px; border-radius: 5px; font-size: 13px; }
        .mail-item { border-bottom: 1px solid #333; padding: 5px 0; color: #ccc; }
        
        #admin-panel { display: none; background: #2d1010; border: 2px dashed red; margin-top: 30px; padding: 20px; }
        
        @media (max-width: 600px) { .grid-container { grid-template-columns: 1fr; } }
    </style>
</head>
<body>

    <h1>🪙 Coin Hunt Pro</h1>

    <div class="grid-container">
        <div class="card">
            <h3>💰 My Wallet</h3>
            <p>Approved: <strong id="db-approved" style="color:var(--gold); font-size: 20px;">0</strong></p>
            <p>Pending: <span id="db-pending" style="color:orange;">0</span></p>
        </div>

        <div class="card">
            <h3>💎 Buy Coins</h3>
            <p style="font-size: 12px;">UPI ID: 6001736850@ybl</p>
            <button class="btn-pay" onclick="openPayment()">Pay Now via UPI</button>
        </div>
    </div>

    <div class="card">
        <canvas id="gameCanvas" width="400" height="200"></canvas>
        <p>Current Score: <b id="current-score">0</b></p>
        <button class="btn-submit" onclick="submitToOwner()">Send Score to Owner for Approval</button>
    </div>

    <div class="card" style="max-width: 900px; margin: auto;">
        <h3>📬 Mailbox</h3>
        <div id="mailbox-messages" class="mailbox">No messages yet...</div>
    </div>

    <button class="btn-admin" onclick="toggleAdmin()">OWNER LOGIN (HIDDEN)</button>

    <div id="admin-panel" class="card">
        <h2 style="color: #ff4444;">🛠️ Owner Dashboard</h2>
        <div id="admin-requests">
            <p>No pending requests from players.</p>
        </div>
    </div>

<script>
    // --- 1. FIREBASE CONFIG (APNA DATA DALEIN) ---
    const firebaseConfig = {
        apiKey: "YOUR_API_KEY",
        authDomain: "YOUR_PROJECT.firebaseapp.com",
        databaseURL: "https://YOUR_PROJECT.firebaseio.com",
        projectId: "YOUR_PROJECT",
        storageBucket: "YOUR_PROJECT.appspot.com",
        messagingSenderId: "YOUR_ID",
        appId: "YOUR_APP_ID"
    };

    firebase.initializeApp(firebaseConfig);
    const db = firebase.database();
    const userId = "Player_01"; // Aap ise dynamic bana sakte hain

    // --- 2. GAME LOGIC ---
    const canvas = document.getElementById('gameCanvas');
    const ctx = canvas.getContext('2d');
    let score = 0, cx = 200, cy = 100;

    function draw() {
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.fillStyle = "gold";
        ctx.beginPath(); ctx.arc(cx, cy, 15, 0, Math.PI*2); ctx.fill();
        ctx.fillStyle = "black"; ctx.fillText("CLICK", cx-15, cy+4);
    }

    canvas.onclick = (e) => {
        score += 10;
        document.getElementById('current-score').innerText = score;
        cx = Math.random() * (canvas.width - 30) + 15;
        cy = Math.random() * (canvas.height - 30) + 15;
        draw();
    };
    draw();

    // --- 3. CORE FUNCTIONS ---

    function openPayment() {
        window.location.href = "upi://pay?pa=6001736850@ybl&pn=CoinGame&cu=INR";
    }

    function submitToOwner() {
        if(score === 0) return alert("Pehle game khelo!");
        db.ref('users/' + userId).once('value').then(snap => {
            let existingPending = (snap.val() && snap.val().pending) ? snap.val().pending : 0;
            db.ref('users/' + userId).update({ pending: existingPending + score });
            score = 0;
            document.getElementById('current-score').innerText = 0;
            alert("Approval request sent to Owner!");
        });
    }

    // Database se data load karna
    db.ref('users/' + userId).on('value', (snap) => {
        const data = snap.val() || { approved: 0, pending: 0 };
        document.getElementById('db-approved').innerText = data.approved;
        document.getElementById('db-pending').innerText = data.pending;
        
        // Admin View Update
        updateAdminView(data.pending);
        
        // Mailbox Update
        const mailDiv = document.getElementById('mailbox-messages');
        if(data.mailbox) {
            mailDiv.innerHTML = Object.values(data.mailbox).reverse().map(m => 
                `<div class="mail-item">📩 ${m.msg} <br><small>${m.date}</small></div>`
            ).join('');
        }
    });

    // --- 4. OWNER/ADMIN LOGIC ---

    function toggleAdmin() {
        const pass = prompt("Enter Owner Password:");
        if(pass === "1234") { // Change this password!
            document.getElementById('admin-panel').style.display = 'block';
        }
    }

    function updateAdminView(pendingAmount) {
        const adminDiv = document.getElementById('admin-requests');
        if(pendingAmount > 0) {
            adminDiv.innerHTML = `
                <p>Player_01 is requesting <b>${pendingAmount} coins</b></p>
                <button onclick="approve(${pendingAmount})" style="background:green; color:white;">APPROVE ✅</button>
                <button onclick="reject()" style="background:red; color:white;">REJECT ❌</button>
            `;
        } else {
            adminDiv.innerHTML = "<p>No pending requests.</p>";
        }
    }

    function approve(amt) {
        db.ref('users/' + userId).once('value').then(snap => {
            let currentApp = snap.val().approved || 0;
            db.ref('users/' + userId).update({
                approved: currentApp + amt,
                pending: 0
            });
            // Send Mail
            db.ref('users/' + userId + '/mailbox').push({
                msg: `Owner approved ${amt} coins. Balance updated!`,
                date: new Date().toLocaleString()
            });
        });
    }

    function reject() {
        db.ref('users/' + userId).update({ pending: 0 });
        db.ref('users/' + userId + '/mailbox').push({
            msg: `Owner rejected your last coin request.`,
            date: new Date().toLocaleString()
        });
    }

</script>
</body>
</html>

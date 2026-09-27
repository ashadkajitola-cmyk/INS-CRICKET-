<script>
    const DEFAULT_ADMIN = "9569981484";
    
    let db = JSON.parse(localStorage.getItem('cricket_pro_db')) || {
        users: {},
        matches: [],
        tickets: [],
        rechargeRequests: [],
        usedTransactions: [],
        settings: {
            adminNumber: DEFAULT_ADMIN,
            maxDevices: 4,
            plans: [
                { id: '4hour', name: '4 Hour Pass', price: 99, durationHours: 4 },
                { id: '1day', name: '1 Day Pass', price: 108, durationHours: 24, matchesCount: 1 },
                { id: '30days', name: '30 Days Monthly Plan', price: 449, durationDays: 30 }
            ]
        },
        deviceSessions: {}
    };

    let currentMobile = localStorage.getItem('cricket_pro_current_mobile') || null;
    let deviceId = localStorage.getItem('cricket_pro_device_id') || 'dev_' + Math.random().toString(36).substring(2,9);
    localStorage.setItem('cricket_pro_device_id', deviceId);

    function saveDB() {
        localStorage.setItem('cricket_pro_db', JSON.stringify(db));
    }

    window.onload = function() {
        if (!db.users[db.settings.adminNumber]) {
            db.users[db.settings.adminNumber] = { wallet: 50000, subscription: null };
        }

        if (currentMobile && db.users[currentMobile]) {
            showDashboard();
        } else {
            currentMobile = null;
            showLogin();
        }
    };

    function showLogin() {
        document.getElementById('loginSection').classList.remove('hidden');
        document.getElementById('mainDashboard').classList.add('hidden');
        document.getElementById('deviceLimitInfo').innerText = `(Max ${db.settings.maxDevices} numbers allowed per device)`;
    }

    function showDashboard() {
        document.getElementById('loginSection').classList.add('hidden');
        document.getElementById('mainDashboard').classList.remove('hidden');
        
        let isAdmin = (currentMobile === db.settings.adminNumber);
        if (isAdmin) {
            document.getElementById('adminCreateMatchBtn').classList.remove('hidden');
            document.getElementById('adminMainControlCard').classList.remove('hidden');
        } else {
            document.getElementById('adminCreateMatchBtn').classList.add('hidden');
            document.getElementById('adminMainControlCard').classList.add('hidden');
        }

        updateHeader();
        renderSubscriptionStatusBox();
        renderSubscriptionPlans();
        renderMatches();
        renderActiveTickets();
        renderMyPurchasedTickets();
    }

    function handleLogin() {
        let mobile = document.getElementById('loginMobileInput').value.trim();
        if (!mobile || mobile.length < 10) {
            alert("Kripya sahi 10-digit mobile number enter karein!");
            return;
        }

        if (!db.deviceSessions[deviceId]) db.deviceSessions[deviceId] = [];
        let activeNumbers = db.deviceSessions[deviceId];
        
        if (!activeNumbers.includes(mobile)) {
            if (activeNumbers.length >= db.settings.maxDevices) {
                alert(`Is device par maximum ${db.settings.maxDevices} numbers hi allow hain!`);
                return;
            }
            activeNumbers.push(mobile);
        }

        if (!db.users[mobile]) {
            let initialWallet = (mobile === db.settings.adminNumber) ? 50000 : 100;
            db.users[mobile] = { wallet: initialWallet, subscription: null };
        }

        currentMobile = mobile;
        localStorage.setItem('cricket_pro_current_mobile', currentMobile);
        saveDB();
        showDashboard();
    }

    function logout() {
        currentMobile = null;
        localStorage.removeItem('cricket_pro_current_mobile');
        showLogin();
    }

    function updateHeader() {
        document.getElementById('displayNumber').innerText = currentMobile;
        let user = db.users[currentMobile];
        document.getElementById('displayWallet').innerText = user ? user.wallet : 0;
        
        if (currentMobile === db.settings.adminNumber) {
            let pending = (db.rechargeRequests || []).filter(r => r.status === 'Pending').length;
            let badge = document.getElementById('pendingCountBadge');
            if (badge) badge.innerText = pending;
        }
    }

    function submitRechargeRequest() {
        let txInput = document.getElementById('txFullInput').value.trim();
        if (!txInput || txInput.length < 6) {
            alert("Kripya valid Transaction ID / UTR enter karein!");
            return;
        }

        if (!db.usedTransactions) db.usedTransactions = [];
        if (db.usedTransactions.includes(txInput)) {
            alert("Yeh Transaction ID pehle hi use ki ja chuki hai!");
            return;
        }

        if (!db.rechargeRequests) db.rechargeRequests = [];
        let existingPending = db.rechargeRequests.find(r => r.txId === txInput && r.status === 'Pending');
        if (existingPending) {
            alert("Is Transaction ID ki request pehle se hi pending hai!");
            return;
        }

        db.rechargeRequests.push({
            id: 'req_' + Date.now(),
            mobile: currentMobile,
            txId: txInput,
            status: 'Pending'
        });

        saveDB();
        document.getElementById('txFullInput').value = '';
        alert("Aapki recharge request bhej di gayi hai!");
    }

    function openRechargeRequestsModal() {
        if (currentMobile !== db.settings.adminNumber) return;
        let listDiv = document.getElementById('pendingRechargesList');
        let pending = (db.rechargeRequests || []).filter(r => r.status === 'Pending');

        if (pending.length === 0) {
            listDiv.innerHTML = '<p>Koi pending recharge request nahi hai.</p>';
        } else {
            let html = '';
            pending.forEach(req => {
                html += `
                    <div class="match-card" style="border-left: 5px solid var(--accent); padding:10px; margin-bottom:10px;">
                        <p><strong>Mobile:</strong> ${req.mobile}</p>
                        <p><strong>Tx ID / UTR:</strong> ${req.txId}</p>
                        <div style="margin-top:8px; display:flex; gap:10px;">
                            <button class="btn btn-success" style="padding:5px 10px; font-size:0.8rem;" onclick="approveRecharge('${req.id}')">Approve & Send ₹210</button>
                            <button class="btn btn-danger" style="padding:5px 10px; font-size:0.8rem;" onclick="rejectRecharge('${req.id}')">Reject</button>
                        </div>
                    </div>
                `;
            });
            listDiv.innerHTML = html;
        }
        document.getElementById('rechargeModal').classList.remove('hidden');
    }

    function closeRechargeModal() { document.getElementById('rechargeModal').classList.add('hidden'); }

    function approveRecharge(reqId) {
        let req = db.rechargeRequests.find(r => r.id === reqId);
        if (!req) return;

        if (!db.usedTransactions) db.usedTransactions = [];
        db.usedTransactions.push(req.txId);
        req.status = 'Approved';

        if (!db.users[req.mobile]) db.users[req.mobile] = { wallet: 0, subscription: null };
        db.users[req.mobile].wallet += 210;

        saveDB();
        updateHeader();
        openRechargeRequestsModal();
        alert("Request approve ho gayi aur ₹210 bhej diye gaye!");
    }

    function rejectRecharge(reqId) {
        db.rechargeRequests = db.rechargeRequests.filter(r => r.id !== reqId);
        saveDB();
        updateHeader();
        openRechargeRequestsModal();
        alert("Request reject kar di gayi.");
    }

    function checkUserHasActiveSub() {
        let isAdmin = (currentMobile === db.settings.adminNumber);
        if (isAdmin) return true;

        let user = db.users[currentMobile];
        if (user && user.subscription) {
            if (user.subscription.matchesLeft !== undefined) {
                return user.subscription.matchesLeft > 0;
            } else {
                return new Date().getTime() < user.subscription.expiresAt;
            }
        }
        return false;
    }

    function renderSubscriptionStatusBox() {
        let box = document.getElementById('activeSubStatusBox');
        let content = document.getElementById('subStatusContent');
        let user = db.users[currentMobile];
        let isAdmin = (currentMobile === db.settings.adminNumber);

        if (isAdmin) {
            box.classList.remove('hidden');
            content.innerHTML = `<strong>Role:</strong> Master Website Admin (Full Access)`;
            return;
        }

        if (user && user.subscription && checkUserHasActiveSub()) {
            box.classList.remove('hidden');
            let sub = user.subscription;
            let now = new Date().getTime();
            let timeLeftText = '';

            let diffMs = sub.expiresAt - now;
            let diffHrs = Math.floor(diffMs / (1000 * 60 * 60));
            let diffDays = Math.floor(diffHrs / 24);
            timeLeftText = diffDays > 0 ? `Validity: ${diffDays} Days (~${diffHrs} Hours)` : `Validity: ${diffHrs} Hours`;

            content.innerHTML = `<strong>Plan:</strong> ${sub.planName} <br>⏱️ ${timeLeftText}`;
        } else {
            box.classList.add('hidden');
        }
    }

    function openCreatorDashboard() {
        let modal = document.getElementById('creatorModal');
        let reqView = document.getElementById('subscriptionRequiredView');
        let actView = document.getElementById('creatorActionView');

        if (checkUserHasActiveSub()) {
            reqView.classList.add('hidden');
            actView.classList.remove('hidden');
        } else {
            reqView.classList.remove('hidden');
            actView.classList.add('hidden');
        }
        modal.classList.remove('hidden');
    }

    function closeCreatorModal() { document.getElementById('creatorModal').classList.add('hidden'); }

    function renderSubscriptionPlans() {
        let listHTML = '';
        db.settings.plans.forEach((plan, index) => {
            let checked = index === 0 ? 'checked' : '';
            listHTML += `<label><input type="radio" name="subPlan" value="${plan.id}" ${checked}> ${plan.name} - ₹${plan.price}</label>`;
        });
        document.getElementById('subPlansList').innerHTML = listHTML;
    }

    function buySelectedSubscription() {
        let selectedRadio = document.querySelector('input[name="subPlan"]:checked');
        if (!selectedRadio) { alert("Kripya pehle koi plan select karein!"); return; }
        let planObj = db.settings.plans.find(p => p.id === selectedRadio.value);
        if (!planObj) return;

        let user = db.users[currentMobile];
        if (user.wallet < planObj.price) {
            alert("Wallet me balance kam hai!");
            return;
        }

        user.wallet -= planObj.price;
        let adminMob = db.settings.adminNumber;
        if (!db.users[adminMob]) db.users[adminMob] = { wallet: 50000, subscription: null };
        db.users[adminMob].wallet += planObj.price;

        let now = new Date().getTime();
        let expiresAt = now + (30 * 24 * 3600 * 1000);
        if (planObj.durationHours) expiresAt = now + (planObj.durationHours * 3600 * 1000);
        if (planObj.durationDays) expiresAt = now + (planObj.durationDays * 24 * 3600 * 1000);

        user.subscription = { planId: planObj.id, planName: planObj.name, expiresAt: expiresAt };

        saveDB();
        updateHeader();
        renderSubscriptionStatusBox();
        closeCreatorModal();
        alert("Subscription successfully buy ho gaya!");
    }

    function openMatchModal() { closeCreatorModal(); document.getElementById('matchModal').classList.remove('hidden'); }
    function closeMatchModal() { document.getElementById('matchModal').classList.add('hidden'); }

    function openTicketCreateModal() {
        closeCreatorModal();
        let select = document.getElementById('ticketMatchSelect');
        let html = '';
        db.matches.forEach(m => {
            html += `<option value="${m.id}">${m.seriesName} (${m.team1} vs ${m.team2})</option>`;
        });
        select.innerHTML = html || '<option value="">Koi match available nahi hai</option>';
        document.getElementById('ticketCreateModal').classList.remove('hidden');
    }
    function closeTicketCreateModal() { document.getElementById('ticketCreateModal').classList.add('hidden'); }

    function saveCustomTicketCreator() {
        let matchId = document.getElementById('ticketMatchSelect').value;
        let price = Number(document.getElementById('customTicketPrice').value);
        let limit = Number(document.getElementById('customTicketLimit').value);

        let match = db.matches.find(m => m.id === matchId);
        if (!match) { alert("Sahi match select karein!"); return; }

        match.price = price;
        match.limit = limit;
        saveDB();
        closeTicketCreateModal();
        renderMatches();
        alert("Tickets successfully create/update kar di gayi is match ke liye!");
    }

    function openInternationalMatchModal() { document.getElementById('internationalMatchModal').classList.remove('hidden'); }
    function closeInternationalMatchModal() { document.getElementById('internationalMatchModal').classList.add('hidden'); }

    function saveMatchSchedule(isInternational) {
        let prefix = isInternational ? 'int' : 'match';
        let matchObj = {
            id: 'match_' + Date.now(),
            creator: currentMobile,
            isInternational: isInternational, // Agar admin ne international modal se banaya hai toh true rahega, warna false
            seriesName: document.getElementById(`${prefix}SeriesName`).value.trim(),
            team1: document.getElementById(`${prefix}Team1`).value.trim(),
            team2: document.getElementById(`${prefix}Team2`).value.trim(),
            score1: document.getElementById(`${prefix}Score1`) ? document.getElementById(`${prefix}Score1`).value.trim() : 'Yet to bat',
            score2: document.getElementById(`${prefix}Score2`) ? document.getElementById(`${prefix}Score2`).value.trim() : 'Yet to bat',
            dateTime: document.getElementById(`${prefix}DateTime`).value.trim() || "2026/23-Sep/3:30 pm",
            venue: document.getElementById(`${prefix}Venue`) ? document.getElementById(`${prefix}Venue`).value.trim() : 'Stadium',
            price: isInternational ? 200 : Number(document.getElementById(`${prefix}Price`) ? document.getElementById(`${prefix}Price`).value : 100),
            limit: isInternational ? 100 : Number(document.getElementById(`${prefix}Limit`) ? document.getElementById(`${prefix}Limit`).value : 50),
            soldCount: 0,
            code6: document.getElementById(`${prefix}Code6`).value.trim(),
            resultStatus: document.getElementById(`${prefix}ResultStatus`).value
        };

        if (!matchObj.seriesName || !matchObj.team1 || !matchObj.team2 || matchObj.code6.length !== 6) {
            alert("Sabhi zaroori fields bharein!");
            return;
        }

        db.matches.push(matchObj);
        saveDB();
        if (isInternational) closeInternationalMatchModal();
        else closeMatchModal();
        renderMatches();
        alert("Match successfully publish ho gaya!");
    }

    function verifyMatchCode() {
        let code = document.getElementById('verifyCodeInput').value.trim();
        let resDiv = document.getElementById('verifyResult');
        let match = db.matches.find(m => m.code6 === code);

        if (!match) {
            resDiv.innerHTML = `<div class="alert-box alert-error">Invalid Code!</div>`;
            return;
        }

        resDiv.innerHTML = `
            <div class="alert-box alert-success">
                <strong>Series:</strong> ${match.seriesName}<br>
                <strong>Match:</strong> ${match.team1} vs ${match.team2}
            </div>
        `;
    }

    function renderMatches() {
        let intList = document.getElementById('internationalMatchesList');
        let apnaList = document.getElementById('apnaMatchesList');
        let intHTML = '', apnaHTML = '';

        db.matches.forEach(match => {
            checkAndSettleMatchResults(match);

            let cardHTML = `
                <div class="match-card">
                    <div class="series-title-bar">
                        <span>${match.seriesName}</span>
                        <span style="font-size:0.75rem; background:#fff; padding:2px 6px; border-radius:4px;">Code: ${match.code6}</span>
                    </div>
                    <div class="match-row">
                        <div class="team-box">🏏 ${match.team1}</div>
                        <div class="vs-text">vs</div>
                        <div class="team-box">${match.team2} 🏏</div>
                    </div>
                    <div class="match-meta">
                        <span>💰 ₹${match.price} (Sold: ${match.soldCount}/${match.limit})</span>
                    </div>
                    <div style="margin-top: 10px; display:flex; justify-content:space-between; align-items:center;">
                        <span style="font-size:0.85rem; font-weight:bold; color:var(--success);">Status: ${match.resultStatus}</span>
                        <div style="display:flex; gap:8px;">
                            <button class="btn btn-success" style="padding:6px 12px; font-size:0.85rem;" onclick="buyTicket('${match.id}')">Buy Ticket</button>
                            ${(currentMobile === db.settings.adminNumber || currentMobile === match.creator) ? 
                                `<button class="btn btn-danger" style="padding:6px 12px; font-size:0.85rem;" onclick="deleteMatch('${match.id}')">Delete</button>` : ''}
                        </div>
                    </div>
                </div>
            `;

            // Agar match international hai toh International tab me dikhega, warna Creator ke hisaab se Apna Schedule me
            if (match.isInternational) {
                intHTML += cardHTML;
            } else if (match.creator === currentMobile) {
                apnaHTML += cardHTML;
            }
        });

        intList.innerHTML = intHTML || '<p>Koi International match nahi hai.</p>';
        apnaList.innerHTML = apnaHTML || '<p>Aapne apna schedule create nahi kiya.</p>';
    }

    function buyTicket(matchId) {
        let match = db.matches.find(m => m.id === matchId);
        if (match.soldCount >= match.limit) { alert("Sold out!"); return; }

        let user = db.users[currentMobile];
        if (user.wallet < match.price) { alert("Balance kam hai!"); return; }

        user.wallet -= match.price;
        let recipientMob = match.creator || db.settings.adminNumber;
        if (!db.users[recipientMob]) db.users[recipientMob] = { wallet: 50000, subscription: null };
        db.users[recipientMob].wallet += match.price;

        match.soldCount += 1;
        db.tickets.push({
            id: 'tkt_' + Date.now(),
            matchId: match.id,
            mobile: currentMobile,
            pricePaid: match.price,
            teamPicked: null,
            sattaAmount: 0,
            status: 'Active'
        });

        saveDB();
        updateHeader();
        renderMatches();
        renderActiveTickets();
        renderMyPurchasedTickets();
        alert("Ticket successfully buy ho gayi!");
    }

    function renderActiveTickets() {
        let listDiv = document.getElementById('activeTicketsList');
        let userTickets = db.tickets.filter(t => t.mobile === currentMobile && t.status === 'Active');

        if (userTickets.length === 0) {
            listDiv.innerHTML = '<p>Koi active running ticket nahi hai.</p>';
            return;
        }

        let html = '';
        userTickets.forEach(tkt => {
            let match = db.matches.find(m => m.id === tkt.matchId);
            if (!match) return;

            html += `
                <div class="match-card" style="border-left: 5px solid var(--success);">
                    <div class="series-title-bar"><span>Ticket ID: ${tkt.id}</span><span>${match.seriesName}</span></div>
                    <p><strong>${match.team1} vs ${match.team2}</strong></p>
                    <div style="margin-top:10px; background:#fcfcfc; padding:12px; border-radius:6px; border:1px solid var(--border);">
                        <h4 style="margin-bottom:8px;">🎲 Satta & Bet Zone</h4>
                        ${tkt.teamPicked ? 
                            `<p>Chuni gayi team: <b>${tkt.teamPicked}</b> | Lagaye gaye ₹: <b>${tkt.sattaAmount}</b></p>` :
                            `<label>Konsi team jeete gi?</label>
                            <select id="sattaTeam_${tkt.id}"><option value="${match.team1}">${match.team1}</option><option value="${match.team2}">${match.team2}</option></select>
                            <label>Bet Amount (₹):</label>
                            <input type="number" id="sattaAmt_${tkt.id}" placeholder="Amount">
                            <button class="btn btn-warning" style="width:100%;" onclick="placeSatta('${tkt.id}')">Confirm Satta</button>`
                        }
                    </div>
                </div>
            `;
        });
        listDiv.innerHTML = html;
    }

    function renderMyPurchasedTickets() {
        let listDiv = document.getElementById('myPurchasedList');
        let userTickets = db.tickets.filter(t => t.mobile === currentMobile);

        if (userTickets.length === 0) {
            listDiv.innerHTML = '<p>Koi ticket history nahi hai.</p>';
            return;
        }

        let html = '';
        userTickets.forEach(tkt => {
            let match = db.matches.find(m => m.id === tkt.matchId);
            let matchName = match ? `${match.team1} vs ${match.team2}` : 'Expired';
            html += `
                <div class="match-card">
                    <p><strong>Ticket ID:</strong> ${tkt.id} | Match: ${matchName}</p>
                    <p style="font-size:0.85rem; color:#666;">Price: ₹${tkt.pricePaid} | Status: ${tkt.status}</p>
                </div>
            `;
        });
        listDiv.innerHTML = html;
    }

    function placeSatta(tktId) {
        let tkt = db.tickets.find(t => t.id === tktId);
        let match = db.matches.find(m => m.id === tkt.matchId);
        let team = document.getElementById(`sattaTeam_${tktId}`).value;
        let amt = Number(document.getElementById(`sattaAmt_${tktId}`).value);

        let user = db.users[currentMobile];
        if (amt <= 0 || user.wallet < amt) { alert("Wallet balance kam hai!"); return; }

        user.wallet -= amt;
        let recipientMob = match.creator || db.settings.adminNumber;
        if (!db.users[recipientMob]) db.users[recipientMob] = { wallet: 50000, subscription: null };
        db.users[recipientMob].wallet += amt;

        tkt.teamPicked = team;
        tkt.sattaAmount = amt;
        saveDB();
        updateHeader();
        renderActiveTickets();
        renderMyPurchasedTickets();
        alert("Satta placed successfully!");
    }

    function checkAndSettleMatchResults(match) {
        if (match.resultStatus === 'Upcoming') return;

        db.tickets.forEach(tkt => {
            if (tkt.matchId === match.id && tkt.status === 'Active' && tkt.teamPicked) {
                let user = db.users[tkt.mobile];
                let recipientMob = match.creator || db.settings.adminNumber;

                if (match.resultStatus.includes('Won')) {
                    let winnerTeam = match.resultStatus.includes('Team 1') ? match.team1 : match.team2;
                    if (tkt.teamPicked === winnerTeam) {
                        let winAmt = tkt.sattaAmount * 2;
                        if (db.users[recipientMob]) db.users[recipientMob].wallet -= winAmt;
                        if (user) user.wallet += winAmt;
                        tkt.status = 'Settled (Won)';
                    } else {
                        tkt.status = 'Settled (Lost)';
                    }
                } else if (match.resultStatus === 'Draw / Abandoned') {
                    if (user) user.wallet += tkt.sattaAmount;
                    tkt.status = 'Refunded';
                }
            }
        });
        saveDB();
    }

    function openMasterAdminPanel() {
        if (currentMobile !== db.settings.adminNumber) return;
        renderAdminAllMatches();
        renderAdminAllTickets();
        renderAdminPlansList();
        document.getElementById('masterAdminModal').classList.remove('hidden');
    }

    function closeMasterAdminModal() { document.getElementById('masterAdminModal').classList.add('hidden'); }

    function switchAdminSubTab(tabName) {
        document.querySelectorAll('.admin-sub-tab').forEach(el => el.classList.add('hidden'));
        if (tabName === 'matches') document.getElementById('adminTabMatches').classList.remove('hidden');
        if (tabName === 'tickets') document.getElementById('adminTabTickets').classList.remove('hidden');
        if (tabName === 'plans') document.getElementById('adminTabPlans').classList.remove('hidden');
    }

    function renderAdminAllMatches() {
        let container = document.getElementById('adminAllMatchesList');
        let html = '';
        db.matches.forEach(match => {
            html += `
                <div class="match-card">
                    <p><strong>${match.seriesName}</strong> (${match.team1} vs ${match.team2}) - ₹${match.price}</p>
                    <div style="margin-top:8px; display:flex; gap:8px;">
                        <button class="btn btn-warning" style="padding:5px 10px; font-size:0.8rem;" onclick="openEditMatchModal('${match.id}')">✏️ Edit Price & Winner</button>
                        <button class="btn btn-danger" style="padding:5px 10px; font-size:0.8rem;" onclick="adminDeleteMatch('${match.id}')">Delete</button>
                    </div>
                </div>
            `;
        });
        container.innerHTML = html || '<p>Koi match nahi hai.</p>';
    }

    function openEditMatchModal(matchId) {
        let match = db.matches.find(m => m.id === matchId);
        if (!match) return;

        document.getElementById('editMatchId').value = match.id;
        document.getElementById('editSeriesName').value = match.seriesName;
        document.getElementById('editTeam1').value = match.team1;
        document.getElementById('editTeam2').value = match.team2;
        document.getElementById('editMatchPrice').value = match.price;
        document.getElementById('editMatchLimit').value = match.limit;
        document.getElementById('editResultStatus').value = match.resultStatus;

        document.getElementById('editSingleMatchModal').classList.remove('hidden');
    }

    function closeEditSingleMatchModal() { document.getElementById('editSingleMatchModal').classList.add('hidden'); }

    function saveEditedMatch() {
        let id = document.getElementById('editMatchId').value;
        let match = db.matches.find(m => m.id === id);
        if (!match) return;

        match.seriesName = document.getElementById('editSeriesName').value.trim();
        match.team1 = document.getElementById('editTeam1').value.trim();
        match.team2 = document.getElementById('editTeam2').value.trim();
        match.price = Number(document.getElementById('editMatchPrice').value);
        match.limit = Number(document.getElementById('editMatchLimit').value);
        match.resultStatus = document.getElementById('editResultStatus').value;

        checkAndSettleMatchResults(match);
        saveDB();
        closeEditSingleMatchModal();
        renderAdminAllMatches();
        renderMatches();
        alert("Match updated successfully!");
    }

    function adminDeleteMatch(matchId) {
        if (confirm("Delete karna chahte hain?")) {
            db.matches = db.matches.filter(m => m.id !== matchId);
            saveDB();
            renderAdminAllMatches();
            renderMatches();
        }
    }

    function renderAdminAllTickets() {
        let container = document.getElementById('adminAllTicketsList');
        let html = '';
        db.tickets.forEach(tkt => {
            html += `
                <div class="match-card">
                    <p><strong>Ticket ID:</strong> ${tkt.id} | User: ${tkt.mobile} | Status: ${tkt.status}</p>
                </div>
            `;
        });
        container.innerHTML = html || '<p>Koi ticket nahi hai.</p>';
    }

    function renderAdminPlansList() {
        let container = document.getElementById('adminPlansList');
        let html = '';
        db.settings.plans.forEach(plan => {
            html += `
                <div class="match-card" style="padding:10px; margin-bottom:8px;">
                    <p><strong>${plan.name}</strong> - ₹${plan.price} (${plan.durationDays ? plan.durationDays + ' Days' : 'Hours'})</p>
                    <button class="btn btn-danger" style="padding:4px 8px; font-size:0.75rem; margin-top:5px;" onclick="deletePlan('${plan.id}')">Delete Plan</button>
                </div>
            `;
        });
        container.innerHTML = html;
    }

    function addNewSubscriptionPlan() {
        let id = document.getElementById('newPlanId').value.trim();
        let name = document.getElementById('newPlanName').value.trim();
        let price = Number(document.getElementById('newPlanPrice').value);
        let days = Number(document.getElementById('newPlanDays').value);

        if (!id || !name || !price) { alert("Saari details bharein!"); return; }

        db.settings.plans.push({ id: id, name: name, price: price, durationDays: days || 30 });
        saveDB();
        renderSubscriptionPlans();
        renderAdminPlansList();
        alert("Naya subscription plan successfully add ho gaya!");
    }

    function deletePlan(planId) {
        db.settings.plans = db.settings.plans.filter(p => p.id !== planId);
        saveDB();
        renderSubscriptionPlans();
        renderAdminPlansList();
        alert("Plan delete ho gaya.");
    }

    function deleteMatch(matchId) {
        if (confirm("Delete karna chahte hain?")) {
            db.matches = db.matches.filter(m => m.id !== matchId);
            saveDB();
            renderMatches();
        }
    }

    function switchTab(tabName) {
        document.querySelectorAll('.tab-content').forEach(el => el.classList.add('hidden'));
        document.querySelectorAll('.tab-btn').forEach(el => el.classList.remove('active'));

        if (tabName === 'international') document.getElementById('internationalTabContent').classList.remove('hidden');
        else if (tabName === 'apna') document.getElementById('apnaTabContent').classList.remove('hidden');
        else if (tabName === 'activeTickets') document.getElementById('activeTicketsTabContent').classList.remove('hidden');
        else if (tabName === 'myPurchased') document.getElementById('myPurchasedTabContent').classList.add ? document.getElementById('myPurchasedTabContent').classList.remove('hidden') : '';
        
        // Fix for myPurchased tab selection visibility
        if (tabName === 'myPurchased') {
            document.getElementById('myPurchasedTabContent').classList.remove('hidden');
        }
        
        event.target.classList.add('active');
    }
</script>

card: { name: 'Банковская карта', icon: '💳', desc: 'Visa, MasterCard, МИР', min: 10, max: 50000, fee: 0 },
        skrill: { name: 'Skrill', icon: '💰', desc: 'Электронный кошелек', min: 5, max: 25000, fee: 0 },
        neteller: { name: 'Neteller', icon: '💎', desc: 'Электронный кошелек', min: 5, max: 25000, fee: 0 },
        perfectmoney: { name: 'Perfect Money', icon: '💵', desc: 'Электронный кошелек', min: 5, max: 25000, fee: 0 },
        webmoney: { name: 'WebMoney', icon: '🟡', desc: 'Электронный кошелек', min: 5, max: 25000, fee: 0 },
        crypto: { name: 'Криптовалюта', icon: '₿', desc: 'BTC, ETH, USDT, BNB', min: 20, max: 100000, fee: 0 }
    };

    function generateCandle() {
        const asset = assets[currentAsset];
        const change = (Math.random() - 0.5) * asset.volatility;
        const newClose = lastPrice + change;
        const bodyHigh = Math.max(lastPrice, newClose);
        const bodyLow = Math.min(lastPrice, newClose);
        const high = bodyHigh + Math.random() * Math.abs(change) * 0.5;
        const low = bodyLow - Math.random() * Math.abs(change) * 0.3;
        const candle = { open: lastPrice, high, low, close: newClose };
        lastPrice = newClose;
        assets[currentAsset].price = newClose;
        return candle;
    }

    function showLoader() {
        const container = document.querySelector('.chart-container');
        if (container && !container.querySelector('.chart-loader')) {
            const loader = document.createElement('div');
            loader.className = 'chart-loader';
            container.appendChild(loader);
        }
    }
    function hideLoader() {
        const loader = document.querySelector('.chart-loader');
        if (loader) loader.remove();
    }

    function initCandles() {
        showLoader();
        setTimeout(() => {
            candleData = [];
            lastPrice = assets[currentAsset].price;
            for (let i = 0; i < candlesCount; i++) candleData.push(generateCandle());
            drawChart();
            updateUI();
            hideLoader();
        }, 50);
    }

    function smoothUpdate() {
        if (animationId) cancelAnimationFrame(animationId);
        const newCandle = generateCandle();
        const oldFirst = { ...candleData[0] };
        const targetFirst = newCandle;
        candleData.shift();
        candleData.push(newCandle);
        let progress = 0;
        function animate() {
            progress += 0.04;
            if (progress >= 1) { drawChart(); updateUI(); animationId = null; return; }
            const ease = 1 - Math.pow(1 - progress, 2);
            const tempData = [...candleData];
            tempData[0] = {
                open: oldFirst.open + (targetFirst.open - oldFirst.open) * ease,
                high: oldFirst.high + (targetFirst.high - oldFirst.high) * ease,
                low: oldFirst.low + (targetFirst.low - oldFirst.low) * ease,
                close: oldFirst.close + (targetFirst.close - oldFirst.close) * ease
            };
            drawChartWithData(tempData);
            animationId = requestAnimationFrame(animate);
        }
        animate();
    }

    function drawChart() { if (canvas && ctx) drawChartWithData(candleData); }
    
    function drawChartWithData(data) {
        if (!canvas || !ctx) return;
        const container = canvas.parentElement;
        const w = container.clientWidth;
        const h = container.clientHeight;
        if (w === 0 || h === 0) return;
        canvas.width = w;
        canvas.height = h;
        ctx.clearRect(0, 0, w, h);
        if (!data.length) return;
        const candleW = Math.max((w - 60) / data.length, 2);
        const startX = 30;
        let minP = Math.min(...data.map(c => c.low));
        let maxP = Math.max(...data.map(c => c.high));
        const pad = (maxP - minP) * 0.1;
        minP -= pad;
        maxP += pad;
        const range = maxP - minP;
        const yPos = (price) => h - 30 - ((price - minP) / range) * (h - 60);
        ctx.strokeStyle = '#1e222d';
        ctx.fillStyle = '#6c7a8e';
        ctx.font = '10px Inter';
        for (let i = 0; i <= 5; i++) {
            const y = h - 30 - (i / 5) * (h - 60);
            ctx.beginPath();
            ctx.moveTo(20, y);
            ctx.lineTo(w - 10, y);
            ctx.stroke();
            const priceValue = minP + (i / 5) * range;
            ctx.fillText(priceValue.toFixed(assets[currentAsset].digits), 5, y - 2);
        }
        for (let i = 0; i < data.length; i++) {
            const c = data[i];
            const x = startX + i * candleW;
            const bodyW = Math.max(candleW * 0.7, 2);
            const wickX = x + bodyW / 2;
            const openY = yPos(c.open);
            const closeY = yPos(c.close);
            const highY = yPos(c.high);
            const lowY = yPos(c.low);
            const isGreen = c.close > c.open;
            ctx.fillStyle = isGreen ? '#00e676' : '#ff5252';
            ctx.strokeStyle = isGreen ? '#00e676' : '#ff5252';
            ctx.beginPath();
            ctx.moveTo(wickX, highY);
            ctx.lineTo(wickX, lowY);
            ctx.stroke();
            const bodyH = Math.abs(closeY - openY);
            const bodyY = Math.min(openY, closeY);
            ctx.fillRect(x, bodyY, bodyW, Math.max(bodyH, 1));
        }
        if (activeTrade) {
            const currentPrice = assets[currentAsset].price;
            const currentY = yPos(currentPrice);
            const startPriceY = yPos(activeTrade.startPrice);
            ctx.beginPath();
            ctx.setLineDash([5, 5]);
            ctx.strokeStyle = activeTrade.direction === 'up' ? '#00e676' : '#ff5252';
            ctx.lineWidth = 2;
            ctx.moveTo(20, startPriceY);
            ctx.lineTo(w - 20, startPriceY);
            ctx.stroke();
            ctx.setLineDash([]);
            const lastCandleX = startX + (data.length - 1) * candleW;
            const lastCandleWickX = lastCandleX + Math.max(candleW * 0.7, 2) / 2;
            ctx.beginPath();
            ctx.fillStyle = activeTrade.direction === 'up' ? '#00e676' : '#ff5252';
            ctx.arc(lastCandleWickX, startPriceY, 6, 0, 2 * Math.PI);
            ctx.fill();
            ctx.fillStyle = '#0a0c12';
            ctx.beginPath();
            ctx.arc(lastCandleWickX, startPriceY, 4, 0, 2 * Math.PI);
            ctx.fill();
        }
    }

    function getCurrentBalance() { return activeAccount === 'demo' ? demoBalance : realBalance; }
    function updateBalanceUI() {
        const balanceEl = document.getElementById('balanceAmount');
        if (balanceEl) balanceEl.textContent = getCurrentBalance().toFixed(2);
        const demoBalanceEl = document.getElementById('demoBalanceAmount');
        const realBalanceEl = document.getElementById('realBalanceAmount');
        if (demoBalanceEl) demoBalanceEl.textContent = demoBalance.toFixed(2);
        if (realBalanceEl) realBalanceEl.textContent = realBalance.toFixed(2);
    }

    function updateUI() {
        const asset = assets[currentAsset];
        const last = candleData[candleData.length - 1];
        if (last) {
            const priceEl = document.getElementById('currentPriceDisplay');
            if (priceEl) priceEl.textContent = last.close.toFixed(asset.digits);
            const titleEl = document.getElementById('currentAssetTitle');
            if (titleEl) titleEl.textContent = currentAsset;
            const change = last.close - last.open;
            const changePercent = (change / last.open) * 100;
            const changeEl = document.getElementById('priceChangeDisplay');
            if (changeEl) {
                if (change > 0) { changeEl.innerHTML = `▲ +${change.toFixed(asset.digits)} (+${changePercent.toFixed(2)}%)`; changeEl.className = 'price-change up'; }
                else if (change < 0) { changeEl.innerHTML = `▼ ${change.toFixed(asset.digits)} (${changePercent.toFixed(2)}%)`; changeEl.className = 'price-change down'; }
                else changeEl.innerHTML = `0.00 (0.00%)`;
            }
        }
        for (let key in assets) {
            const el = document.getElementById(`price_${key.replace('/', '')}`);
            if (el) el.textContent = assets[key].price.toFixed(assets[key].digits);
        }
        updateBalanceUI();
        const indicator = document.getElementById('activeTradeIndicator');
        if (indicator && activeTrade) {
            indicator.style.display = 'block';
            indicator.className = `active-trade-indicator ${activeTrade.direction === 'up' ? 'up' : 'down'}`;
            const remaining = Math.ceil((activeTrade.expiryTime - Date.now()) / 1000);
            indicator.innerHTML = `${activeTrade.direction === 'up' ? '📈 СТАВКА ВВЕРХ' : '📉 СТАВКА ВНИЗ'} | ${activeTrade.amount}$ | Осталось: ${remaining} сек`;
        } else if (indicator) indicator.style.display = 'none';
    }

    function selectAsset(name) {
        if (animationId) cancelAnimationFrame(animationId);
        currentAsset = name;
        lastPrice = assets[currentAsset].price;
        initCandles();
        document.querySelectorAll('.asset-row').forEach(el => {
            if (el.dataset.asset === name) el.classList.add('active');
            else el.classList.remove('active');
        });
        updateUI();
    }

    function makeTrade(direction) {
        if (activeTrade) {
            document.getElementById('tradeStatus').innerHTML = '⏳ Дождитесь завершения текущей сделки!';
            return;
        }
        const amount = parseFloat(document.getElementById('tradeAmount').value);
        const expirySec = parseInt(document.getElementById('expiryTime').value);
        const currentBalance = getCurrentBalance();
        if (isNaN(amount) || amount <= 0 || amount > currentBalance) {
            document.getElementById('tradeStatus').innerHTML = `⚠️ Неверная сумма или недостаточно средств на ${activeAccount === 'demo' ? 'демо' : 'реальном'} счете!`;
            return;
        }
        
        if (activeAccount === 'demo') demoBalance -= amount;
        else realBalance -= amount;
        updateBalanceUI();
        
        const startPrice = assets[currentAsset].price;
        activeTrade = {
            direction, amount, startPrice, expiryTime: Date.now() + expirySec * 1000,
            asset: currentAsset, expirySec, account: activeAccount
        };
        
        playTrade();
        document.getElementById('tradeStatus').innerHTML = `📊 СДЕЛКА ${direction === 'up' ? 'ВВЕРХ ⬆️' : 'ВНИЗ ⬇️'} на ${amount}$ | Экспирация: ${expirySec} сек | ${activeAccount === 'demo' ? 'ДЕМО' : 'РЕАЛ'}`;
        drawChart();
        
        const interval = setInterval(() => {
            if (!activeTrade) { clearInterval(interval); return; }
            if (Date.now() >= activeTrade.expiryTime) {
                clearInterval(interval);
                const endPrice = assets[currentAsset].price;
                const win = activeTrade.direction === 'up' ? endPrice > activeTrade.startPrice : endPrice < activeTrade.startPrice;
                const profit = win ? activeTrade.amount * 0.85 : -activeTrade.amount;
                
                if (activeTrade.account === 'demo') demoBalance += activeTrade.amount + profit;
                else realBalance += activeTrade.amount + profit;
                updateBalanceUI();
                
                if (win) { playWin(); showToast(`🎉 Победа! +${profit.toFixed(2)}$`, 'win'); }
                else { playLoss(); showToast(`💔 Поражение! ${profit.toFixed(2)}$`, 'loss'); }
                
                if (currentUser) {
                    const userData = JSON.parse(localStorage.getItem(`user_${currentUser.login}`) || '{}');
                    const history = userData.history || [];
                    history.unshift({
                        asset: activeTrade.asset, dir: activeTrade.direction, amount: activeTrade.amount,
                        expiry: activeTrade.expirySec + 'с', result: win ? 'win' : 'loss', profit,
                        time: new Date().toLocaleTimeString(), account: activeTrade.account
                    });
                    if (history.length > 50) history.pop();
                    userData.history = history;
                    userData.demoBalance = demoBalance;
                    userData.realBalance = realBalance;
                    localStorage.setItem(`user_${currentUser.login}`, JSON.stringify(userData));
                }
                renderMainHistory();
                renderProfileHistory();
                document.getElementById('tradeStatus').innerHTML = win ? `✅ ПОБЕДА! +${profit.toFixed(2)}$` : `❌ ПОРАЖЕНИЕ! ${profit.toFixed(2)}$`;
                activeTrade = null;
                drawChart();
                setTimeout(() => {
                    if (document.getElementById('tradeStatus').innerHTML.includes('ПОБЕДА') || document.getElementById('tradeStatus').innerHTML.includes('ПОРАЖЕНИЕ'))
                        document.getElementById('tradeStatus').innerHTML = '✅ Готов к торговле';
                }, 2500);
            } else {
                const rem = Math.ceil((activeTrade.expiryTime - Date.now()) / 1000);
                document.getElementById('tradeStatus').innerHTML = `⏱️ ${activeTrade.direction === 'up' ? '📈 ВВЕРХ' : '📉 ВНИЗ'} ${activeTrade.amount}$ | Осталось: ${rem} сек | ${activeTrade.account === 'demo' ? 'ДЕМО' : 'РЕАЛ'}`;
                updateUI();
                drawChart();
            }
        }, 100);
        updateUI();
        drawChart();
    }

    function switchAccount(account) {
        activeAccount = account;
        const demoBtn = document.getElementById('demoAccountBtn');
        const realBtn = document.getElementById('realAccountBtn');
        if (demoBtn) demoBtn.classList.toggle('active', account === 'demo');
        if (realBtn) realBtn.classList.toggle('active', account === 'real');
        updateBalanceUI();
        const statusEl = document.getElementById('tradeStatus');
        if (statusEl) statusEl.innerHTML = `✅ Готов к торговле (${account === 'demo' ? 'ДЕМО-счет' : 'РЕАЛЬНЫЙ счет'})`;
    }

    function showDepositModal() { /* (сохранен без изменений) */ 
        let selectedMethod = 'card';
        const modalHtml = `<div class="modal" id="depositModal"><div class="modal-content"><h2><i class="fas fa-credit-card"></i> Пополнение счета</h2><div class="payment-methods" id="paymentMethodsList">${Object.entries(paymentMethods).map(([key, method]) => `<div class="payment-method" data-method="${key}"><div class="payment-icon">${method.icon}</div><div class="payment-info"><div class="payment-name">${method.name}</div><div class="payment-desc">${method.desc}</div><div class="payment-limit">от ${method.min}$ до ${method.max}$</div></div></div>`).join('')}</div><div id="paymentDetails"></div><div class="preset-amounts"><button class="preset-amount" data-amount="50">50$</button><button class="preset-amount" data-amount="100">100$</button><button class="preset-amount" data-amount="500">500$</button><button class="preset-amount" data-amount="1000">1000$</button><button class="preset-amount" data-amount="5000">5000$</button></div><input type="number" id="depositAmountInput" class="amount-input" value="100" min="1" step="10"><button id="confirmDepositBtn" class="deposit-btn">Пополнить</button><button id="closeDepositBtn" class="secondary-btn">Отмена</button><div id="depositMessage" class="error-msg"></div><div class="info-text"><i class="fas fa-shield-alt"></i> Безопасные платежи. Комиссия не взимается.</div></div></div>`;
        document.body.insertAdjacentHTML('beforeend', modalHtml);
        const updatePaymentDetails = (methodKey) => {
            const method = paymentMethods[methodKey];
            const detailsDiv = document.getElementById('paymentDetails');
            if (methodKey === 'crypto') {
                detailsDiv.innerHTML = `<div class="crypto-address"><strong>BTC:</strong> 1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa<br><strong>ETH:</strong> 0x742d35Cc6634C0532925a3b844Bc9e7598eFfCdB<br><strong>USDT (TRC20):</strong> TQYxZq8Zq8Zq8Zq8Zq8Zq8Zq8Zq8Zq8Zq8<br><strong>BNB (BEP20):</strong> bnb1q8zq8zq8zq8zq8zq8zq8zq8zq8zq8zq8zq8</div><button class="copy-btn" onclick="navigator.clipboard.writeText('1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa')">📋 Скопировать BTC адрес</button>`;
            } else detailsDiv.innerHTML = `<div style="background:#0a0c12;border-radius:12px;padding:16px;margin:16px 0;"><div>Сумма: <strong>${parseFloat(document.getElementById('depositAmountInput')?.value||100)}$</strong></div><div class="info-text">🔐 После нажатия "Пополнить" вы будете перенаправлены на оплату</div></div>`;
        };
        document.querySelectorAll('.payment-method').forEach(el => el.addEventListener('click', () => { document.querySelectorAll('.payment-method').forEach(m => m.classList.remove('selected')); el.classList.add('selected'); selectedMethod = el.dataset.method; updatePaymentDetails(selectedMethod); }));
        document.querySelector('.payment-method').classList.add('selected'); updatePaymentDetails('card');
        document.getElementById('confirmDepositBtn').onclick = () => {
            const amount = parseFloat(document.getElementById('depositAmountInput').value);
            const method = paymentMethods[selectedMethod];
            if (isNaN(amount) || amount <= 0 || amount < method.min || amount > method.max) { document.getElementById('depositMessage').textContent = `Сумма должна быть от ${method.min}$ до ${method.max}$`; return; }
            realBalance += amount; updateBalanceUI();
            if (currentUser) { const userData = JSON.parse(localStorage.getItem(`user_${currentUser.login}`) || '{}'); userData.realBalance = realBalance; localStorage.setItem(`user_${currentUser.login}`, JSON.stringify(userData)); }
            document.getElementById('depositMessage').textContent = `✅ Счет пополнен на ${amount}$!`; document.getElementById('depositMessage').style.color = '#00e676';
            setTimeout(() => { const modal = document.getElementById('depositModal'); if(modal) modal.remove(); }, 1500);
        };
        document.getElementById('closeDepositBtn').onclick = () => { const modal = document.getElementById('depositModal'); if(modal) modal.remove(); };
    }

    function renderMainHistory() {
        const history = currentUser ? (JSON.parse(localStorage.getItem(`user_${currentUser.login}`) || '{}').history || []) : [];
        const container = document.getElementById('historyList');
        if (!container) return;
        if (!history.length) { container.innerHTML = '<div style="text-align: center; padding: 24px; color: #5a6a85;">Нет сделок</div>'; return; }
        container.innerHTML = history.map(h => `<div class="history-item"><span><strong>${h.asset}</strong> ${h.dir === 'up' ? '⬆️' : '⬇️'} ${h.amount}$ (${h.expiry}) ${h.account === 'demo' ? '🎮' : '💰'}</span><span class="${h.result === 'win' ? 'history-win' : 'history-loss'}">${h.result === 'win' ? '+' : ''}${h.profit.toFixed(2)}$</span><span style="color:#6c7a8e;">${h.time}</span></div>`).join('');
    }
    function renderProfileHistory() {
        const history = currentUser ? (JSON.parse(localStorage.getItem(`user_${currentUser.login}`) || '{}').history || []) : [];
        const container = document.getElementById('profileHistoryList');
        if (!container) return;
        if (!history.length) { container.innerHTML = '<div style="text-align: center; padding: 16px; color: #5a6a85;">Нет сделок</div>'; return; }
        container.innerHTML = history.map(h => `<div class="profile-history-item"><span><strong>${h.asset}</strong> ${h.dir === 'up' ? '⬆️' : '⬇️'} ${h.amount}$ ${h.account === 'demo' ? '(ДЕМО)' : '(РЕАЛ)'}</span><span class="${h.result === 'win' ? 'history-win' : 'history-loss'}">${h.result === 'win' ? '+' : ''}${h.profit.toFixed(2)}$</span><span style="color:#6c7a8e;">${h.time}</span></div>`).join('');
    }

    function logout() {
        if (currentUser) {
            const userData = JSON.parse(localStorage.getItem(`user_${currentUser.login}`) || '{}');
            userData.demoBalance = demoBalance; userData.realBalance = realBalance;
            localStorage.setItem(`user_${currentUser.login}`, JSON.stringify(userData));
        }
        currentUser = null;
        showAuthModal();
    }

    function setupCanvasResize() {
        if (!canvas) return;
        const resizeHandler = () => { if (canvas && canvas.parentElement) drawChart(); };
        if (resizeObserver) resizeObserver.disconnect();
        resizeObserver = new ResizeObserver(() => requestAnimationFrame(resizeHandler));
        resizeObserver.observe(canvas.parentElement);
        window.addEventListener('resize', resizeHandler);
    }

    // СТАРАЯ СТИЛЬНАЯ МОДАЛКА
    function showAuthModal() {
        const authHtml = `
            <div class="auth-overlay" id="authOverlay">
                <div class="auth-card">
                    <div class="auth-header">
                        <div class="auth-logo"><i class="fas fa-chart-line"></i> Trayd</div>
                        <div class="auth-subtitle">Профессиональная торговая платформа</div>
                    </div>
                    <div id="authLoginForm">
                        <input type="text" id="loginUsername" class="auth-input" placeholder="Логин">
                        <input type="password" id="loginPassword" class="auth-input" placeholder="Пароль">
                        <button id="loginSubmitBtn" class="auth-btn">Войти в аккаунт</button>
                        <div class="auth-switch">Нет аккаунта? <span id="showRegisterBtn">Зарегистрироваться</span></div>
                        <div id="authErrorMsg" class="auth-error"></div>
                    </div>
                    <div id="authRegisterForm" style="display: none;">
                        <input type="text" id="regEmail" class="auth-input" placeholder="Email">
                        <input type="text" id="regName" class="auth-input" placeholder="Имя">
                        <input type="tel" id="regPhone" class="auth-input" placeholder="Телефон">
                        <input type="text" id="regLogin" class="auth-input" placeholder="Логин">
                        <input type="password" id="regPassword" class="auth-input" placeholder="Пароль">
                        <button id="registerSubmitBtn" class="auth-btn">Создать аккаунт</button>
                        <div class="auth-switch">Уже есть аккаунт? <span id="showLoginBtn">Войти</span></div>
                        <div id="regErrorMsg" class="auth-error"></div>
                    </div>
                    <div class="auth-divider">ДЕМО-ДОСТУП</div>
                    <button id="demoAccessBtn" class="auth-btn" style="background: #2a2f3e; color: #00e676;">🎮 Продолжить без регистрации (ДЕМО)</button>
                </div>
            </div>
        `;
        document.getElementById('app').innerHTML = authHtml;
        
        const loginFormDiv = document.getElementById('authLoginForm');
        const registerFormDiv = document.getElementById('authRegisterForm');
        const authError = document.getElementById('authErrorMsg');
        const regError = document.getElementById('regErrorMsg');
        
        document.getElementById('showRegisterBtn').onclick = () => { loginFormDiv.style.display = 'none'; registerFormDiv.style.display = 'block'; authError.textContent = ''; regError.textContent = ''; };
        document.getElementById('showLoginBtn').onclick = () => { registerFormDiv.style.display = 'none'; loginFormDiv.style.display = 'block'; authError.textContent = ''; regError.textContent = ''; };
        
        document.getElementById('loginSubmitBtn').onclick = () => {
            const login = document.getElementById('loginUsername').value.trim();
            const password = document.getElementById('loginPassword').value;
            if (!login || !password) { authError.textContent = 'Введите логин и пароль'; return; }
            const userData = localStorage.getItem(`user_${login}`);
            if (userData) {
                const data = JSON.parse(userData);
                if (data.password === password) {
                    currentUser = { login, password, name: data.name, email: data.email, phone: data.phone };
                    demoBalance = data.demoBalance !== undefined ? data.demoBalance : 10000;
                    realBalance = data.realBalance !== undefined ? data.realBalance : 0;
                    loadMainApp();
                } else authError.textContent = 'Неверный пароль';
            } else authError.textContent = 'Пользователь не найден';
        };
        
        document.getElementById('registerSubmitBtn').onclick = () => {
            const email = document.getElementById('regEmail').value.trim();
            const name = document.getElementById('regName').value.trim();
            const phone = document.getElementById('regPhone').value.trim();
            const login = document.getElementById('regLogin').value.trim();
            const password = document.getElementById('regPassword').value;
            if (!email || !name || !phone || !login || !password) { regError.textContent = 'Заполните все поля'; return; }
            if (!email.includes('@')) { regError.textContent = 'Введите корректный email'; return; }
            if (phone.length < 10) { regError.textContent = 'Введите корректный телефон'; return; }
            if (localStorage.getItem(`user_${login}`)) { regError.textContent = 'Пользователь с таким логином уже существует'; return; }
            const newUser = { login, password, name, email, phone, demoBalance: 10000, realBalance: 0, history: [], registeredAt: new Date().toISOString() };
            localStorage.setItem(`user_${login}`, JSON.stringify(newUser));
            currentUser = { login, password, name, email, phone };
            demoBalance = 10000; realBalance = 0;
            loadMainApp();
        };
        
        document.getElementById('demoAccessBtn').onclick = () => {
            currentUser = { login: 'demo_user', name: 'Гость', email: 'demo@trayd.com', phone: 'Демо-режим' };
            demoBalance = 10000; realBalance = 0;
            loadMainApp();
        };
    }

    function loadMainApp() {
        const appHtml = `
            <div class="top-bar"><div class="logo"><i class="fas fa-chart-line"></i> Trayd</div>
                <div class="balance-panel"><div class="balance-item"><i class="fas fa-wallet"></i> ${activeAccount === 'demo' ? 'ДЕМО-счет' : 'РЕАЛЬНЫЙ счет'}</div>
                <div class="balance-item"><span class="balance-amount" id="balanceAmount">${getCurrentBalance().toFixed(2)}</span> USD</div>
                <div class="account-switch"><button class="account-btn demo active" id="demoAccountBtn">🎮 ДЕМО</button><button class="account-btn real" id="realAccountBtn">💰 РЕАЛ</button></div></div>
                <div class="profile-section"><button class="notification-btn" id="notificationBtn"><i class="fas fa-bell"></i></button>
                <div class="profile-menu"><div class="profile-trigger" id="profileTrigger"><div class="avatar">${(currentUser.name || currentUser.login).charAt(0).toUpperCase()}</div><span class="profile-name">${currentUser.name || currentUser.login}</span><i class="fas fa-chevron-down"></i></div>
                <div class="dropdown-menu" id="dropdownMenu"><div class="dropdown-header"><div class="dropdown-name">${currentUser.name || currentUser.login}</div><div class="dropdown-email">${currentUser.email || currentUser.login + '@trayd.com'}</div><div style="margin-top:12px; padding-top:8px; border-top:1px solid #2a2f3e;"><div style="display:flex; justify-content:space-between;"><span>🎮 Демо:</span><span id="demoBalanceAmount">${demoBalance.toFixed(2)}</span></div><div style="display:flex; justify-content:space-between;"><span>💰 Реал:</span><span id="realBalanceAmount">${realBalance.toFixed(2)}</span></div></div></div>
                <div class="dropdown-item" id="depositBtn"><i class="fas fa-plus-circle"></i> Пополнить реальный счет</div>
                <div class="dropdown-item" id="profileHistoryBtn"><i class="fas fa-history"></i> История сделок</div>
                <div class="dropdown-divider"></div><div class="dropdown-item logout-item" id="logoutBtn"><i class="fas fa-sign-out-alt"></i> Выйти</div></div></div></div></div>
            <div class="app-container"><div class="assets-panel"><div class="assets-header"><span><i class="fas fa-chart-line"></i> Активы</span></div>
                <div class="asset-category"><div class="category-title">FOREX</div><div class="asset-row" data-asset="EUR/USD"><span class="asset-name">EUR/USD</span><span class="asset-price" id="price_EURUSD">1.09250</span></div>
                <div class="asset-row" data-asset="GBP/USD"><span class="asset-name">GBP/USD</span><span class="asset-price" id="price_GBPUSD">1.28430</span></div>
                <div class="asset-row" data-asset="USD/JPY"><span class="asset-name">USD/JPY</span><span class="asset-price" id="price_USDJPY">148.25</span></div></div>
                <div class="asset-category"><div class="category-title">CRYPTO</div><div class="asset-row" data-asset="BTC/USD"><span class="asset-name">BTC/USD</span><span class="asset-price" id="price_BTCUSD">42580</span></div>
                <div class="asset-row" data-asset="ETH/USD"><span class="asset-name">ETH/USD</span><span class="asset-price" id="price_ETHUSD">2280</span></div></div>
                <div class="asset-category"><div class="category-title">COMMODITIES</div><div class="asset-row" data-asset="Gold"><span class="asset-name">Gold</span><span class="asset-price" id="price_Gold">2045.30</span></div></div>
                <div class="asset-category"><div class="category-title">STOCKS</div><div class="asset-row" data-asset="Apple"><span class="asset-name">Apple</span><span class="asset-price" id="price_Apple">175.20</span></div>
                <div class="asset-row" data-asset="Tesla"><span class="asset-name">Tesla</span><span class="asset-price" id="price_Tesla">238.45</span></div></div></div>
                <div class="chart-panel"><div class="chart-header"><div><h2 id="currentAssetTitle">EUR/USD</h2><div><span class="current-price" id="currentPriceDisplay">1.09250</span><span class="price-change" id="priceChangeDisplay"></span></div></div>
                <div class="timeframe-group"><button class="tf-btn active" data-candles="20">20</button><button class="tf-btn" data-candles="30">30</button><button class="tf-btn" data-candles="50">50</button><button class="tf-btn" data-candles="100">100</button></div></div>
                <div class="chart-container"><canvas id="candleCanvas"></canvas><div id="activeTradeIndicator" class="active-trade-indicator" style="display: none;"></div></div></div>
                <div class="trade-panel"><div class="trade-tabs"><div class="tab-btn active">Бинарные опционы</div><div class="tab-btn">Цифровой опцион</div></div>
                <div class="trade-content"><div class="amount-card"><div class="amount-label">Сумма сделки (USD)</div><input type="number" id="tradeAmount" class="trade-amount-input" value="100" step="10" min="1" max="10000">
                <div class="preset-buttons"><button class="preset" data-amount="10">10</button><button class="preset" data-amount="50">50</button><button class="preset" data-amount="100">100</button><button class="preset" data-amount="500">500</button><button class="preset" data-amount="1000">1000</button></div></div>
                <div class="expiry-card"><div class="amount-label">Время экспирации</div><select id="expiryTime" class="expiry-select"><option value="3">⚡ 3 секунды</option><option value="5">🔥 5 секунд</option><option value="15">15 секунд</option><option value="30">30 секунд</option><option value="60" selected>1 минута</option><option value="120">2 минуты</option><option value="300">5 минут</option></select></div>
                <div class="trade-buttons"><button class="btn-call" id="callBtn"><i class="fas fa-arrow-up"></i> ВВЕРХ</button><button class="btn-put" id="putBtn"><i class="fas fa-arrow-down"></i> ВНИЗ</button></div>
                <div class="status-card" id="tradeStatus">✅ Готов к торговле (ДЕМО)</div></div>
                <div class="history-section"><div class="history-title"><i class="fas fa-clock"></i> История сделок</div><div id="historyList"><div style="text-align: center; padding: 24px; color: #5a6a85;">Нет сделок</div></div></div></div></div>
            <div class="modal" id="profileHistoryModal" style="display: none;"><div class="modal-content" style="width: 500px; max-height: 80vh; overflow-y: auto;"><h2><i class="fas fa-history"></i> История сделок</h2><div id="profileHistoryList" style="max-height: 400px; overflow-y: auto;"></div><button id="closeHistoryModal" style="margin-top: 20px; background: #2a2f3e;">Закрыть</button></div></div>
        `;
        document.getElementById('app').innerHTML = appHtml;
        canvas = document.getElementById('candleCanvas');
        ctx = canvas.getContext('2d');
        setupCanvasResize();
        initCandles();
        renderMainHistory();
        
        document.getElementById('callBtn').onclick = () => makeTrade('up');
        document.getElementById('putBtn').onclick = () => makeTrade('down');
        document.querySelectorAll('.preset').forEach(btn => btn.onclick = () => document.getElementById('tradeAmount').value = btn.dataset.amount);
        document.querySelectorAll('.tf-btn').forEach(btn => { btn.onclick = () => { document.querySelectorAll('.tf-btn').forEach(b => b.classList.remove('active')); btn.classList.add('active'); candlesCount = parseInt(btn.dataset.candles); initCandles(); }; });
        document.querySelectorAll('.asset-row').forEach(el => el.onclick = () => selectAsset(el.dataset.asset));
        document.getElementById('demoAccountBtn').onclick = () => switchAccount('demo');
        document.getElementById('realAccountBtn').onclick = () => switchAccount('real');
        const profileTrigger = document.getElementById('profileTrigger');
        const dropdownMenu = document.getElementById('dropdownMenu');
        profileTrigger.onclick = (e) => { e.stopPropagation(); dropdownMenu.classList.toggle('show'); };
        document.addEventListener('click', () => dropdownMenu.classList.remove('show'));
        document.getElementById('logoutBtn').onclick = () => logout();
        document.getElementById('depositBtn').onclick = () => showDepositModal();
        const profileHistoryModal = document.getElementById('profileHistoryModal');
        document.getElementById('profileHistoryBtn').onclick = () => { renderProfileHistory(); profileHistoryModal.style.display = 'flex'; };
        document.getElementById('closeHistoryModal').onclick = () => profileHistoryModal.style.display = 'none';
        document.getElementById('notificationBtn').onclick = () => showToast('📢 Новых уведомлений нет', 'info');
        setInterval(() => smoothUpdate(), 3000);
    }
    showAuthModal();
</script>
</body>
</html>

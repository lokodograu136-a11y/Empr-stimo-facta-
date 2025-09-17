<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>EmprestaFácil Pro</title>
    <!-- Firebase -->
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-app-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-firestore-compat.js"></script>
    <script src="https://www.gstatic.com/firebasejs/10.12.2/firebase-auth-compat.js"></script>
    <style>
        /* ... (mesmo CSS da versão anterior) ... */
        /* ADICIONE ESTE ESTILO NO FINAL DO SEU CSS */
        .login-container {
            max-width: 400px;
            margin: 100px auto;
            padding: 2rem;
            background: white;
            border-radius: 12px;
            box-shadow: 0 4px 20px rgba(0,0,0,0.1);
        }
        .login-container h2 {
            text-align: center;
            margin-bottom: 1.5rem;
            color: #2563eb;
        }
        .paid-btn {
            background: #10b981;
            padding: 6px 12px;
            font-size: 0.85rem;
            margin-left: 10px;
        }
        .paid-btn:hover {
            background: #059669;
        }
        .filter-bar {
            display: flex;
            gap: 10px;
            margin-bottom: 1rem;
            flex-wrap: wrap;
        }
    </style>
</head>
<body>
    <!-- TELA DE LOGIN -->
    <div id="login-screen" class="container" style="display: flex; justify-content: center; align-items: center; min-height: 100vh;">
        <div class="login-container">
            <h2>🔐 Acesso Restrito</h2>
            <div class="form-group">
                <label for="email">E-mail</label>
                <input type="email" id="email" placeholder="seu@email.com" required>
            </div>
            <div class="form-group">
                <label for="password">Senha</label>
                <input type="password" id="password" placeholder="••••••••" required>
            </div>
            <button id="login-btn" style="width: 100%;">Entrar</button>
            <p style="text-align: center; margin-top: 1rem; font-size: 0.9rem;">
                Esqueceu a senha? <a href="#" id="reset-password">Clique aqui</a>
            </p>
        </div>
    </div>

    <!-- CONTEÚDO PRINCIPAL (ESCONDIDO ATÉ LOGIN) -->
    <div id="main-content" style="display: none;">
        <!-- ... (todo o HTML da versão anterior aqui) ... -->
        <!-- ADICIONE ESTE BOTÃO NA LISTA DE EMPRÉSTIMOS -->
        <!-- Dentro do updateLoansList(), substitua o innerHTML do item por: -->
    </div>

    <script>
        // Configuração do Firebase — SUBSTITUA PELOS SEUS DADOS
        const firebaseConfig = {
            apiKey: "SUA_API_KEY",
            authDomain: "SEU_PROJETO.firebaseapp.com",
            projectId: "SEU_PROJETO",
            storageBucket: "SEU_PROJETO.appspot.com",
            messagingSenderId: "SEU_ID",
            appId: "SEU_APP_ID"
        };

        // Inicializa Firebase
        firebase.initializeApp(firebaseConfig);
        const db = firebase.firestore();
        const auth = firebase.auth();

        let currentUser = null;
        let clients = [];
        let loans = [];

        // Elementos
        const loginScreen = document.getElementById('login-screen');
        const mainContent = document.getElementById('main-content');
        const loginBtn = document.getElementById('login-btn');
        const resetPassword = document.getElementById('reset-password');

        // Login
        loginBtn.addEventListener('click', async () => {
            const email = document.getElementById('email').value;
            const password = document.getElementById('password').value;

            try {
                const userCredential = await auth.signInWithEmailAndPassword(email, password);
                currentUser = userCredential.user;
                loginScreen.style.display = 'none';
                mainContent.style.display = 'block';
                loadUserData();
                alert('Login realizado com sucesso!');
            } catch (error) {
                alert('Erro no login: ' + error.message);
            }
        });

        // Recuperar senha
        resetPassword.addEventListener('click', async () => {
            const email = prompt('Digite seu e-mail para recuperar a senha:');
            if (email) {
                try {
                    await auth.sendPasswordResetEmail(email);
                    alert('E-mail de recuperação enviado!');
                } catch (error) {
                    alert('Erro: ' + error.message);
                }
            }
        });

        // Carregar dados do usuário logado
        async function loadUserData() {
            if (!currentUser) return;

            // Carregar clientes
            const clientsSnapshot = await db.collection(`users/${currentUser.uid}/clients`).get();
            clients = clientsSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

            // Carregar empréstimos
            const loansSnapshot = await db.collection(`users/${currentUser.uid}/loans`).get();
            loans = loansSnapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));

            updateClientSelect();
            updateDashboard();
            updateClientsList();
            updateLoansList();
        }

        // Salvar cliente no Firebase
        async function saveClient(client) {
            const docRef = await db.collection(`users/${currentUser.uid}/clients`).add(client);
            client.id = docRef.id;
            clients.push(client);
        }

        // Salvar empréstimo no Firebase
        async function saveLoan(loan) {
            const docRef = await db.collection(`users/${currentUser.uid}/loans`).add(loan);
            loan.id = docRef.id;
            loans.push(loan);
        }

        // Marcar empréstimo como pago
        async function markAsPaid(loanId) {
            await db.collection(`users/${currentUser.uid}/loans`).doc(loanId).update({
                status: 'paid',
                paidAt: new Date().toISOString()
            });

            // Atualizar localmente
            const loan = loans.find(l => l.id === loanId);
            if (loan) loan.status = 'paid';
            updateDashboard();
            updateLoansList();
        }

        // Atualizar lista de empréstimos — com botão de "Marcar como Pago"
        function updateLoansList() {
            const list = document.getElementById('loans-list');
            if (loans.length === 0) {
                list.innerHTML = '<p>Nenhum empréstimo registrado ainda.</p>';
                return;
            }

            list.innerHTML = `
                <div class="filter-bar">
                    <select id="filter-status">
                        <option value="all">Todos</option>
                        <option value="unpaid">Pendentes</option>
                        <option value="paid">Pagos</option>
                    </select>
                    <input type="date" id="filter-date" placeholder="Filtrar por data">
                </div>
            `;

            const filtered = loans.filter(loan => {
                const statusFilter = document.getElementById('filter-status').value;
                const dateFilter = document.getElementById('filter-date').value;
                if (statusFilter !== 'all' && loan.status !== statusFilter) return false;
                if (dateFilter && loan.dueDate !== dateFilter) return false;
                return true;
            });

            filtered.slice().reverse().forEach(loan => {
                const div = document.createElement('div');
                div.className = 'loan-item';
                div.innerHTML = `
                    <div class="loan-header">
                        <strong>${loan.clientName}</strong>
                        <span class="status ${loan.status}">${loan.status === 'paid' ? 'Pago' : 'Pendente'}</span>
                        ${loan.status !== 'paid' ? 
                            `<button class="paid-btn" onclick="markAsPaid('${loan.id}')">✅ Pagar</button>` : ''}
                    </div>
                    <div>💵 Emprestado: R$ ${loan.amount.toFixed(2).replace('.', ',')}</div>
                    <div>📈 A receber: R$ ${loan.amountToReceive.toFixed(2).replace('.', ',')}</div>
                    <div>📅 Vencimento: ${new Date(loan.dueDate).toLocaleDateString('pt-BR')}</div>
                    <div>📊 Juros: ${loan.rate}%</div>
                `;
                list.appendChild(div);
            });
        }

        // ... (RESTANTE DAS FUNÇÕES DA VERSÃO ANTERIOR) ...

        // Substitua as funções de cadastro para usar Firebase:
        clientForm.addEventListener('submit', async function(e) {
            e.preventDefault();
            const name = document.getElementById('client-name').value;
            const phone = document.getElementById('client-phone').value;
            const business = document.getElementById('client-business').value;
            
            const client = {
                name,
                phone,
                business,
                createdAt: new Date().toISOString()
            };
            
            await saveClient(client);
            clientForm.reset();
            updateClientSelect();
            updateClientsList();
            updateDashboard();
            alert('Cliente cadastrado com sucesso!');
        });

        loanForm.addEventListener('submit', async function(e) {
            e.preventDefault();
            const clientId = parseInt(clientSelect.value);
            const amount = parseFloat(loanAmount.value);
            const rate = parseFloat(interestRate.value);
            const dueDate = document.getElementById('due-date').value;
            
            const client = clients.find(c => c.id === clientId);
            if (!client) {
                alert('Selecione um cliente válido!');
                return;
            }
            
            const loan = {
                clientId,
                clientName: client.name,
                amount,
                rate,
                dueDate,
                amountToReceive: amount * (1 + rate / 100),
                status: 'unpaid',
                createdAt: new Date().toISOString()
            };
            
            await saveLoan(loan);
            loanForm.reset();
            amountToReceive.value = '';
            updateDashboard();
            updateLoansList();
            alert('Empréstimo registrado com sucesso!');
        });

        // Torna a função markAsPaid global para o onclick funcionar
        window.markAsPaid = markAsPaid;
    </script>
</body>
</html>

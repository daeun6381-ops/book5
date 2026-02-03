# book5
<!DOCTYPE html>
<html lang="ko">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>2026 사각사각 - Paper Journal</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css" rel="stylesheet">
    
    <!-- Firebase SDK 임포트 -->
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-app.js";
        import { getFirestore, collection, doc, setDoc, deleteDoc, onSnapshot, query, orderBy, serverTimestamp } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-firestore.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.0.1/firebase-auth.js";

        // 전역 설정 로드
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const db = getFirestore(app);
        const auth = getAuth(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

        window.db = db;
        window.auth = auth;
        window.appId = appId;

        // 데이터 수신 리스너 (한 번만 실행되도록 관리)
        let unsubscribe = null;

        function startListening(user) {
            if (!user || !db) return;
            
            // 기존 리스너가 있다면 제거
            if (unsubscribe) unsubscribe();

            const q = query(
                collection(db, 'artifacts', appId, 'public', 'data', 'journals'),
                orderBy("createdAt", "desc")
            );

            unsubscribe = onSnapshot(q, (snapshot) => {
                const books = [];
                snapshot.forEach((doc) => {
                    books.push({ id: doc.id, ...doc.data() });
                });
                window.updateBooksFromServer(books);
            }, (error) => {
                console.error("데이터 읽기 오류:", error);
                // 에러 발생 시 사용자에게 알림
                const list = document.getElementById('book-list');
                if (list) {
                    list.innerHTML = `<div class="col-span-full text-center py-40"><p class="gamja-flower text-xl text-red-400">데이터를 불러올 수 없습니다. 다시 시도해 주세요.</p></div>`;
                }
            });
        }

        // 인증 로직 실행
        const initAuth = async () => {
            try {
                // 커스텀 토큰이 있으면 우선 사용, 없으면 익명 로그인
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("로그인 실패:", error);
            }
        };

        // 초기화 시작
        onAuthStateChanged(auth, (user) => {
            if (user) {
                window.currentUser = user;
                startListening(user);
            } else {
                initAuth();
            }
        });

        // 서버 저장 함수
        window.saveBookToServer = async (book) => {
            if (!auth.currentUser || !db) return;
            try {
                const bookRef = doc(db, 'artifacts', appId, 'public', 'data', 'journals', String(book.id));
                await setDoc(bookRef, {
                    ...book,
                    updatedAt: serverTimestamp(),
                    createdAt: book.createdAt || serverTimestamp()
                }, { merge: true });
            } catch (e) {
                console.error("저장 오류:", e);
            }
        };

        // 서버 삭제 함수
        window.deleteBookFromServer = async (id) => {
            if (!auth.currentUser || !db) return;
            try {
                const bookRef = doc(db, 'artifacts', appId, 'public', 'data', 'journals', String(id));
                await deleteDoc(bookRef);
            } catch (e) {
                console.error("삭제 오류:", e);
            }
        };
    </script>

    <style>
        @import url('https://fonts.googleapis.com/css2?family=Gamja+Flower&family=Noto+Serif+KR:wght@300;400;700&display=swap');
        
        :root {
            --paper-color: #ffffff;
            --line-color: #f0f0f0;
            --deep-pink: #ff8fa3;
            --star-yellow: #f1c40f;
            --text-main: #2d3436;
            --bg-color: #f4f4f4;
        }

        body {
            font-family: 'Noto Serif KR', serif;
            background-color: var(--bg-color);
            color: var(--text-main);
            margin: 0;
            display: flex;
            justify-content: center;
            min-height: 100vh;
            -webkit-tap-highlight-color: transparent;
        }

        .notebook {
            width: 100%;
            max-width: 1800px;
            min-height: 100vh;
            background-color: var(--paper-color);
            background-image: linear-gradient(var(--line-color) 1px, transparent 1px);
            background-size: 100% 3rem;
            box-shadow: 0 0 30px rgba(0,0,0,0.05);
            position: relative;
            padding-bottom: 120px;
            margin: 0;
            border-radius: 0;
            border: none;
        }

        @media (min-width: 768px) {
            .notebook {
                margin: 20px;
                width: calc(100% - 40px);
                border-radius: 24px;
                border: 1px solid #e0e0e0;
            }
        }

        .gamja-flower {
            font-family: 'Gamja Flower', cursive;
        }

        header {
            padding: 3rem 0 1.5rem;
            text-align: center;
        }

        header h1 {
            font-family: 'Gamja Flower', cursive;
            font-size: clamp(2.5rem, 8vw, 4rem);
            opacity: 0.9;
            color: var(--text-main);
        }

        #book-list {
            padding: 0 2rem;
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
            gap: 1.5rem;
            align-content: start;
        }

        @media (min-width: 1440px) {
            #book-list {
                grid-template-columns: repeat(auto-fill, minmax(420px, 1fr));
                gap: 2.5rem;
                padding: 0 4rem;
            }
        }

        .book-tab {
            background: white;
            border: 1px solid #eee;
            border-radius: 20px;
            padding: 1.5rem;
            display: flex;
            gap: 1.2rem;
            position: relative;
            box-shadow: 6px 6px 20px rgba(0,0,0,0.02);
            touch-action: manipulation;
        }

        .cover-slot {
            width: 90px;
            height: 126px;
            background-color: #fafafa;
            border: 1px dashed #ddd;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            overflow: hidden;
            flex-shrink: 0;
            border-radius: 12px;
        }

        @media (min-width: 768px) {
            .cover-slot { width: 100px; height: 140px; }
        }

        .cover-slot img { width: 100%; height: 100%; object-fit: cover; }

        .star-rating {
            display: flex;
            align-items: center;
            gap: 4px;
            padding: 4px 0;
        }

        .star-rating i {
            font-size: 1.4rem;
            color: #eee;
            cursor: pointer;
        }

        .star-rating i.active { color: var(--star-yellow); }

        .review-input {
            width: 100%;
            border: none;
            outline: none;
            background: transparent;
            resize: none;
            font-size: 1.5rem;
            color: #444;
            margin-top: 0.5rem;
            font-family: 'Gamja Flower', cursive;
            min-height: 110px;
            line-height: 1.5;
            padding-bottom: 40px; 
        }

        .delete-btn {
            position: absolute;
            bottom: 0.5rem;
            right: 0.5rem;
            color: #ddd;
            cursor: pointer;
            width: 48px;
            height: 48px;
            display: flex;
            align-items: center;
            justify-content: center;
            z-index: 10;
            background: transparent;
            border-radius: 50%;
            transition: all 0.2s;
        }

        #custom-confirm {
            position: fixed;
            top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.5);
            display: none;
            justify-content: center;
            align-items: center;
            z-index: 2000;
            backdrop-filter: blur(4px);
            padding: 20px;
        }

        .modal-content {
            background: white;
            padding: 2rem;
            border-radius: 24px;
            width: 100%;
            max-width: 320px;
            text-align: center;
            box-shadow: 0 20px 40px rgba(0,0,0,0.2);
        }

        .modal-btns {
            display: flex;
            gap: 12px;
            margin-top: 1.8rem;
        }

        .modal-btns button {
            flex: 1;
            padding: 14px;
            border-radius: 15px;
            font-family: 'Gamja Flower', cursive;
            font-size: 1.3rem;
            cursor: pointer;
        }

        .btn-cancel { background: #f5f5f5; color: #666; }
        .btn-confirm { background: var(--deep-pink); color: white; }

        .fab {
            position: fixed;
            bottom: 2rem;
            right: 2rem; 
            width: 70px;
            height: 70px;
            background-color: var(--deep-pink);
            color: white;
            border-radius: 50%;
            display: flex;
            align-items: center;
            justify-content: center;
            box-shadow: 0 8px 25px rgba(255, 143, 163, 0.4);
            cursor: pointer;
            z-index: 100;
        }

        .pagination {
            display: flex;
            justify-content: center;
            align-items: center;
            gap: 0.8rem;
            margin-top: 4rem;
            padding-bottom: 3rem;
            flex-wrap: wrap;
        }

        .page-btn, .nav-btn {
            min-width: 48px;
            height: 48px;
            border-radius: 14px;
            display: flex;
            align-items: center;
            justify-content: center;
            cursor: pointer;
            font-family: 'Gamja Flower', cursive;
            font-size: 1.5rem;
            background: white;
            border: 1px solid #eee;
            color: #888;
        }

        .page-btn.active {
            background-color: var(--text-main);
            color: white;
            border-color: var(--text-main);
        }
    </style>
</head>
<body>

    <div class="notebook">
        <header>
            <h1 id="main-title" class="gamja-flower">2026 사각사각-1</h1>
        </header>

        <main>
            <div id="book-list">
                <div class="col-span-full text-center py-40 opacity-20">
                    <p class="gamja-flower" style="font-size:1.5rem">서재 문을 열고 있어요...</p>
                </div>
            </div>
            <div id="pagination" class="pagination"></div>
        </main>
    </div>

    <div id="custom-confirm" onclick="if(event.target === this) closeModal()">
        <div class="modal-content">
            <p class="gamja-flower text-2xl mb-2">정말로 삭제할까요?</p>
            <p class="text-gray-400 text-sm mb-4">기록된 문장은 복구할 수 없어요.</p>
            <div class="modal-btns">
                <button class="btn-cancel" onclick="closeModal()">취소</button>
                <button class="btn-confirm" id="confirm-delete-btn">삭제</button>
            </div>
        </div>
    </div>

    <div class="fab" onclick="addNewBookTab()" title="새 기록 추가">
        <i class="fas fa-plus text-2xl"></i>
    </div>

    <script>
        let allBooks = []; 
        let currentPage = 1;
        const itemsPerPage = 8;
        let pendingDeleteId = null;

        // 서버에서 데이터를 받았을 때 실행되는 함수
        window.updateBooksFromServer = (books) => {
            allBooks = books;
            const maxPage = Math.max(1, Math.ceil(allBooks.length / itemsPerPage));
            if (currentPage > maxPage) currentPage = maxPage;
            renderBooks();
            renderPagination();
            updateTitle();
        };

        function updateTitle() {
            const titleEl = document.getElementById('main-title');
            if (titleEl) titleEl.innerText = `2026 사각사각-${currentPage}`;
        }

        function changePage(page) {
            const maxPage = Math.ceil(allBooks.length / itemsPerPage);
            if (page < 1 || page > maxPage) return;
            currentPage = page;
            renderBooks();
            renderPagination();
            updateTitle();
            window.scrollTo({ top: 0, behavior: 'smooth' });
        }

        function addNewBookTab() {
            const id = Date.now();
            const newBook = {
                id: id,
                cover: '',
                rating: 0,
                review: '',
                date: new Date().toLocaleDateString('ko-KR'),
                createdAt: new Date().getTime()
            };
            if(window.saveBookToServer) window.saveBookToServer(newBook);
        }

        function setRating(id, rating) {
            const book = allBooks.find(b => b.id == id);
            if (book) {
                book.rating = rating;
                window.saveBookToServer(book);
            }
        }

        function updateReview(id, text) {
            const book = allBooks.find(b => b.id == id);
            if (book && book.review !== text) {
                book.review = text;
                window.saveBookToServer(book);
            }
        }

        async function handleImageUpload(id, event) {
            const file = event.target.files[0];
            if (!file) return;
            const reader = new FileReader();
            reader.onload = async (e) => {
                const img = new Image();
                img.src = e.target.result;
                img.onload = () => {
                    const canvas = document.createElement('canvas');
                    const ctx = canvas.getContext('2d');
                    canvas.width = 600; canvas.height = 840;
                    ctx.drawImage(img, 0, 0, 600, 840);
                    const book = allBooks.find(b => b.id == id);
                    if (book) {
                        book.cover = canvas.toDataURL('image/jpeg', 0.8);
                        window.saveBookToServer(book);
                    }
                };
            };
            reader.readAsDataURL(file);
        }

        function openDeleteModal(id) {
            pendingDeleteId = id;
            document.getElementById('custom-confirm').style.display = 'flex';
        }

        function closeModal() {
            pendingDeleteId = null;
            document.getElementById('custom-confirm').style.display = 'none';
        }

        document.getElementById('confirm-delete-btn').addEventListener('click', () => {
            if (pendingDeleteId && window.deleteBookFromServer) {
                window.deleteBookFromServer(pendingDeleteId);
                closeModal();
            }
        });

        function renderBooks() {
            const list = document.getElementById('book-list');
            if (!list) return;

            if (allBooks.length === 0) {
                list.innerHTML = `<div class="col-span-full text-center py-40 opacity-20"><p class="gamja-flower text-3xl">비어있는 서재입니다.</p></div>`;
                return;
            }
            
            const startIndex = (currentPage - 1) * itemsPerPage;
            const paginatedBooks = allBooks.slice(startIndex, startIndex + itemsPerPage);
            
            list.innerHTML = paginatedBooks.map(book => `
                <div class="book-tab" id="tab-${book.id}">
                    <div class="cover-slot" onclick="document.getElementById('file-${book.id}').click()">
                        ${book.cover ? `<img src="${book.cover}" alt="Cover">` : `<i class="fas fa-camera text-gray-200 text-2xl"></i>`}
                        <input type="file" id="file-${book.id}" class="hidden" accept="image/*" onchange="handleImageUpload('${book.id}', event)">
                    </div>
                    <div class="flex-grow w-full relative">
                        <div class="flex flex-col sm:flex-row sm:justify-between sm:items-center mb-1">
                            <div class="star-rating">
                                ${[1, 2, 3, 4, 5].map(num => `<i class="fas fa-star ${num <= book.rating ? 'active' : ''}" onclick="setRating('${book.id}', ${num})"></i>`).join('')}
                            </div>
                            <span class="gamja-flower text-gray-400" style="font-size:1.1rem">${book.date}</span>
                        </div>
                        <textarea class="review-input" 
                                  placeholder="이 책의 문장을 기록하세요..." 
                                  onblur="updateReview('${book.id}', this.value)">${book.review || ''}</textarea>
                        
                        <div class="delete-btn" onclick="openDeleteModal('${book.id}')">
                            <i class="fas fa-trash-alt"></i>
                        </div>
                    </div>
                </div>
            `).join('');
        }

        function renderPagination() {
            const paginationContainer = document.getElementById('pagination');
            if (!paginationContainer) return;

            const totalPages = Math.ceil(allBooks.length / itemsPerPage);
            if (totalPages <= 1) { paginationContainer.innerHTML = ''; return; }
            
            let html = `<button class="nav-btn" onclick="changePage(${currentPage - 1})" ${currentPage === 1 ? 'disabled' : ''}><i class="fas fa-chevron-left"></i></button>`;
            for (let i = 1; i <= totalPages; i++) {
                html += `<div class="page-btn ${i === currentPage ? 'active' : ''}" onclick="changePage(${i})">${i}</div>`;
            }
            html += `<button class="nav-btn" onclick="changePage(${currentPage + 1})" ${currentPage === totalPages ? 'disabled' : ''}><i class="fas fa-chevron-right"></i></button>`;
            paginationContainer.innerHTML = html;
        }
    </script>
</body>
</html>

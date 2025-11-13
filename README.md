<!DOCTYPE html>
<html lang="es">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>La Movida Noticias</title>
    <!-- Carga de Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Carga de Firebase (Módulos ES) -->
    <script type="module">
        // Importar funciones necesarias de Firebase
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { 
            getAuth, 
            onAuthStateChanged, 
            signInWithEmailAndPassword, 
            signOut,
            signInAnonymously,
            signInWithCustomToken,
            setPersistence,
            inMemoryPersistence
        } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { 
            getFirestore, 
            collection, 
            addDoc, 
            query, 
            where, 
            onSnapshot, 
            serverTimestamp,
            doc,
            setDoc,
            getDoc
        } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // --- Configuración de Firebase ---
        const firebaseConfig = JSON.parse(typeof __firebase_config !== 'undefined' ? __firebase_config : '{}');
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';
        const initialAuthToken = typeof __initial_auth_token !== 'undefined' ? __initial_auth_token : null;

        let app, auth, db, userId;
        let articlesCollection;

        // Elementos del DOM
        let nacionalFeed, internacionalFeed, adminPanel, loginForm, publishForm, logoutButton, messageBox;
        let navNacional, navInternacional, navAdmin;

        // --- Inicialización de la App ---
        function initializeAppAndFirebase() {
            try {
                app = initializeApp(firebaseConfig);
                auth = getAuth(app);
                db = getFirestore(app);
                console.log("Firebase inicializado correctamente.");
                authListener();
            } catch (error) {
                console.error("Error al inicializar Firebase:", error);
                // Asegurarse de que messageBox exista antes de usarlo
                if (messageBox) {
                    messageBox.textContent = "Error crítico al cargar la aplicación.";
                    messageBox.className = "p-3 rounded-md bg-red-100 text-red-700 mb-4";
                }
            }
        }

        // --- Manejador de Autenticación ---
        async function authListener() {
            try {
                await setPersistence(auth, inMemoryPersistence);
                if (initialAuthToken) {
                    await signInWithCustomToken(auth, initialAuthToken);
                } else {
                    await signInAnonymously(auth);
                }
            } catch (error) {
                console.error("Error durante el inicio de sesión inicial:", error);
            }

            onAuthStateChanged(auth, (user) => {
                if (user && !user.isAnonymous) {
                    // Usuario "autorizado" (ha iniciado sesión con email/pass)
                    userId = user.uid;
                    console.log("Usuario autorizado conectado:", userId);
                    loginForm.classList.add('hidden');
                    publishForm.classList.remove('hidden');
                    logoutButton.classList.remove('hidden');
                    adminPanel.classList.remove('hidden');
                } else {
                    // Usuario anónimo o no logueado
                    userId = user ? user.uid : 'anonymous-' + crypto.randomUUID();
                    console.log("Usuario anónimo o desconectado.");
                    loginForm.classList.remove('hidden');
                    publishForm.classList.add('hidden');
                    logoutButton.classList.add('hidden');
                    // No ocultamos el adminPanel si el usuario hace clic en "Admin"
                }

                // Definir la ruta de la colección pública de artículos
                const articlesCollectionPath = `/artifacts/${appId}/public/data/articles`;
                articlesCollection = collection(db, articlesCollectionPath);
                
                // Cargar artículos una vez que la autenticación está lista
                setupArticleListeners(); // Nueva función que carga todas las secciones
            });
        }

        // --- Lógica de Navegación y Carga de Noticias ---
        function setupNavigation() {
            // Los enlaces de navegación ahora solo apuntan a anclas (href="#nacional-section")
            // por lo que no necesitan JS, excepto el botón de Admin.

            navAdmin.addEventListener('click', (e) => {
                e.preventDefault();
                adminPanel.classList.toggle('hidden');
            });
        }

        // --- Cargar Artículos (Firestore) ---
        function setupArticleListeners() {
            if (!articlesCollection) {
                console.warn("La colección de artículos aún no está lista.");
                return;
            }

            // Listener para Noticias Nacionales
            const qNacional = query(articlesCollection, where("section", "==", "nacional"));
            onSnapshot(qNacional, (querySnapshot) => {
                const articles = [];
                querySnapshot.forEach((doc) => {
                    articles.push({ id: doc.id, ...doc.data() });
                });

                // Ordenar en JavaScript
                articles.sort((a, b) => (b.createdAt?.toMillis() || 0) - (a.createdAt?.toMillis() || 0));

                // Renderizar en la sección 'Nacional' con layout de 'héroe'
                renderArticles(articles, nacionalFeed, 'hero');

            }, (error) => {
                console.error("Error al cargar noticias nacionales: ", error);
                nacionalFeed.innerHTML = "<p class='text-red-600'>No se pudieron cargar las noticias.</p>";
            });

            // Listener para Noticias Internacionales
            const qInternacional = query(articlesCollection, where("section", "==", "internacional"));
            onSnapshot(qInternacional, (querySnapshot) => {
                const articles = [];
                querySnapshot.forEach((doc) => {
                    articles.push({ id: doc.id, ...doc.data() });
                });

                // Ordenar en JavaScript
                articles.sort((a, b) => (b.createdAt?.toMillis() || 0) - (a.createdAt?.toMillis() || 0));

                // Renderizar en la sección 'Internacional' con layout de 'grid'
                renderArticles(articles, internacionalFeed, 'grid');

            }, (error) => {
                console.error("Error al cargar noticias internacionales: ", error);
                internacionalFeed.innerHTML = "<p class='text-red-600'>No se pudieron cargar las noticias.</p>";
            });
        }

        // --- Renderizar Artículos en el DOM ---
        function renderArticles(articles, element, layoutType) {
            element.innerHTML = ''; // Limpiar noticias anteriores
            if (articles.length === 0) {
                element.innerHTML = "<p class='text-gray-600 col-span-full text-center'>No hay noticias en esta sección todavía.</p>";
                return;
            }

            // Limitar el número de artículos mostrados
            const articlesToRender = layoutType === 'hero' 
                ? articles.slice(0, 5) // 1 héroe + 4 pequeños
                : articles.slice(0, 4); // 4 pequeños

            articlesToRender.forEach((article, index) => {
                const articleEl = document.createElement('div');
                
                // Acortar el contenido para un snippet
                const snippet = article.content.length > 100 
                    ? article.content.substring(0, 100) + '...' 
                    : article.content;
                
                // Fecha legible
                const date = article.createdAt?.toDate().toLocaleString('es-ES', { dateStyle: 'medium', timeStyle: 'short' }) || 'Fecha no disponible';

                // Definir clases basadas en el layout
                const isHero = (layoutType === 'hero' && index === 0);
                
                let containerClasses = "bg-white rounded-lg shadow-lg overflow-hidden flex flex-col transition-transform duration-300 hover:scale-105";
                let imgClasses = "w-full h-48 object-cover";
                let titleClasses = "text-2xl font-bold text-gray-900 mb-2";
                let contentClasses = "p-6 flex-grow";

                if (isHero) {
                    // Clases para el artículo Héroe (más grande)
                    containerClasses += " lg:col-span-2 lg:row-span-2";
                    imgClasses = "w-full h-64 md:h-96 object-cover";
                    titleClasses = "text-3xl md:text-4xl font-bold text-gray-900 mb-3";
                    contentClasses = "p-6 md:p-8 flex-grow";
                }

                articleEl.className = containerClasses;

                articleEl.innerHTML = `
                    <img src="${article.imageUrl || 'https://placehold.co/600x400/cccccc/ffffff?text=Noticia'}" 
                         alt="${article.title}" 
                         class="${imgClasses}"
                         onerror="this.src='https://placehold.co/600x400/cccccc/ffffff?text=Error+Img';">
                    <div class="${contentClasses}">
                        <h3 class="${titleClasses}">${article.title}</h3>
                        <p class="text-gray-700 mb-4">${snippet}</p>
                    </div>
                    <div class="p-6 pt-0 mt-auto">
                        <span class="text-sm text-gray-500">${date}</span>
                        <a href="#" class="text-red-700 hover:text-red-900 font-semibold ml-4 float-right">Leer más &rarr;</a>
                    </div>
                `;
                element.appendChild(articleEl);
            });
        }

        // --- Lógica de Administración (Login, Logout, Publish) ---
        function setupAdminActions() {
            // Login
            loginForm.addEventListener('submit', async (e) => {
                e.preventDefault();
                let email = loginForm.email.value; // Cambiado a 'let'
                const password = loginForm.password.value;
                
                // --- INICIO DE LA MODIFICACIÓN ---
                // Esta es la lógica "híbrida" que solicitaste.
                // El email real en Firebase DEBE ser 'admin@lamovida.com'
                // (o el que definas abajo), pero el usuario solo necesita
                // escribir 'admin' en el formulario.

                // ¡IMPORTANTE! Este debe ser el email que creaste en Firebase Auth.
                const hardcodedAdminEmail = "admin@lamovida.com";
                
                if (email === "admin" && password === "maluma99") {
                    console.log("Detectado inicio de sesión de 'admin'. Usando email real de Firebase.");
                    email = hardcodedAdminEmail; 
                }
                // --- FIN DE LA MODIFICACIÓN ---

                try {
                    await signInWithEmailAndPassword(auth, email, password);
                    showMessage("Inicio de sesión exitoso.", "success");
                } catch (error) {
                    console.error("Error de inicio de sesión:", error);
                    showMessage(`Error: ${error.message}`, "error");
                }
            });

            // Logout
            logoutButton.addEventListener('click', async () => {
                try {
                    await signOut(auth);
                    showMessage("Sesión cerrada.", "success");
                    // El authListener se encargará de ocultar el formulario de publicación
                    // Volvemos a firmar anónimamente para que pueda seguir viendo noticias
                    await authListener(); 
                } catch (error) {
                    console.error("Error al cerrar sesión:", error);
                    showMessage(`Error: ${error.message}`, "error");
                }
            });

            // Publicar Noticia
            publishForm.addEventListener('submit', async (e) => {
                e.preventDefault();
                if (!articlesCollection) {
                    showMessage("Error: La base de datos no está lista.", "error");
                    return;
                }

                const title = publishForm.title.value;
                const content = publishForm.content.value;
                const section = publishForm.section.value;
                const imageUrl = publishForm.imageUrl.value;

                if (!title || !content || !section) {
                    showMessage("Por favor, completa todos los campos obligatorios.", "error");
                    return;
                }

                try {
                    await addDoc(articlesCollection, {
                        title: title,
                        content: content,
                        section: section,
                        imageUrl: imageUrl,
                        createdAt: serverTimestamp(),
                        authorId: userId // Guardar quién lo publicó
                    });
                    
                    showMessage("¡Noticia publicada con éxito!", "success");
                    publishForm.reset();

                } catch (error) {
                    console.error("Error al publicar la noticia:", error);
                    showMessage(`Error al publicar: ${error.message}`, "error");
                }
            });
        }

        // --- Utilidad de Mensajes ---
        function showMessage(message, type = "success") {
            messageBox.textContent = message;
            if (type === "error") {
                messageBox.className = "p-3 rounded-md bg-red-100 text-red-700 mb-4";
            } else {
                messageBox.className = "p-3 rounded-md bg-green-100 text-green-700 mb-4";
            }
            // Ocultar mensaje después de 5 segundos
            setTimeout(() => {
                messageBox.textContent = '';
                messageBox.className = '';
            }, 5000);
        }


        // --- Ejecución al Cargar la Página ---
        document.addEventListener('DOMContentLoaded', () => {
            // Obtener referencias a elementos clave
            nacionalFeed = document.getElementById('nacional-feed');
            internacionalFeed = document.getElementById('internacional-feed');
            adminPanel = document.getElementById('admin-panel');
            loginForm = document.getElementById('login-form');
            publishForm = document.getElementById('publish-form');
            logoutButton = document.getElementById('logout-button');
            messageBox = document.getElementById('message-box');
            navNacional = document.getElementById('nav-nacional');
            navInternacional = document.getElementById('nav-internacional');
            navAdmin = document.getElementById('nav-admin');

            // Configurar la navegación
            setupNavigation();
            
            // Configurar acciones de administrador (Restaurado)
            setupAdminActions();

            // Iniciar Firebase (Restaurado)
            initializeAppAndFirebase();
        });

    </script>
    
    <!-- Estilos personalizados (complemento de Tailwind) -->
    <!-- Este bloque <style> se ha restaurado y colocado en el lugar correcto -->
    <style>
        /* Estilo para fuente (Inter) */
        @import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;600;700;900&display=swap');
        body {
            font-family: 'Inter', sans-serif;
        }
        /* Color principal inspirado en portales de noticias (rojo oscuro) */
        .bg-brand-red {
            background-color: #990000; /* Un rojo periodístico, como el de El País o El Universal */
        }
        .text-brand-red {
            color: #990000;
        }
        .border-brand-red {
            border-color: #990000;
        }
    </style>
</head>
<body class="bg-gray-100">

    <!-- Encabezado y Navegación -->
    <header class="bg-brand-red text-white shadow-lg sticky top-0 z-50">
        <div class="container mx-auto px-4 py-4">
            <div class="flex justify-between items-center">
                <!-- Logo/Título -->
                <h1 class="text-4xl font-black tracking-tight">
                    La Movida Noticias
                </h1>
                
                <!-- Navegación Principal -->
                <nav class="hidden md:flex space-x-6 items-center">
                    <a id="nav-nacional" href="#nacional-section" class="text-lg font-semibold text-white hover:text-gray-200 pb-1 transition duration-300">Nacional</a>
                    <a id="nav-internacional" href="#internacional-section" class="text-lg font-semibold text-gray-300 hover:text-white pb-1 transition duration-300">Internacional</a>
                </nav>

                <!-- Acceso Admin -->
                <div>
                    <a id="nav-admin" href="#" class="text-sm font-medium bg-white text-brand-red px-4 py-2 rounded-full hover:bg-gray-200 transition duration-300">
                        Administrar
                    </a>
                </div>
            </div>
        </div>
    </header>

    <!-- Contenido Principal -->
    <main class="container mx-auto px-4 mt-8">

        <!-- Panel de Administración (Oculto por defecto) -->
        <section id="admin-panel" class="hidden bg-white p-6 md:p-8 rounded-xl shadow-2xl mb-12 border-t-4 border-brand-red">
            <h3 class="text-3xl font-bold text-gray-800 mb-6">Panel de Administración</h3>

            <!-- Contenedor de Mensajes -->
            <div id="message-box" class="mb-4"></div>

            <!-- Formulario de Login -->
            <form id="login-form" class="space-y-4">
                <p class="text-gray-600">Inicia sesión para publicar noticias.</p>
                <div>
                    <!-- Cambio de Label para que acepte un nombre de usuario -->
                    <label for="email" class="block text-sm font-medium text-gray-700">Usuario o Email</label>
                    <!-- Cambio de 'type' a 'text' para permitir 'admin' -->
                    <input type="text" id="email" name="email" required class="mt-1 block w-full md:w-1/2 px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red">
                </div>
                <div>
                    <label for="password" class="block text-sm font-medium text-gray-700">Contraseña</label>
                    <input type="password" id="password" name="password" required class="mt-1 block w-full md:w-1/2 px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red">
                </div>
                <button type="submit" class="px-6 py-2 bg-brand-red text-white font-semibold rounded-lg shadow-md hover:bg-red-800 transition duration-300">
                    Iniciar Sesión
                </button>
            </form>

            <!-- Formulario de Publicación (Oculto hasta iniciar sesión) -->
            <form id="publish-form" class="hidden space-y-6">
                <h4 class="text-2xl font-semibold text-gray-700">Publicar Nueva Noticia</h4>
                <div>
                    <label for="title" class="block text-sm font-medium text-gray-700">Título</label>
                    <input type="text" id="title" name="title" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red">
                </div>
                <div>
                    <label for="imageUrl" class="block text-sm font-medium text-gray-700">URL de la Imagen (Opcional)</label>
                    <input type="url" id="imageUrl" name="imageUrl" class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red">
                </div>
                <div>
                    <label for="section" class="block text-sm font-medium text-gray-700">Sección</label>
                    <select id="section" name="section" required class="mt-1 block w-full md:w-1/3 px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red">
                        <option value="nacional">Nacional</option>
                        <option value="internacional">Internacional</option>
                    </select>
                </div>
                <div>
                    <label for="content" class="block text-sm font-medium text-gray-700">Contenido de la Noticia</label>
                    <textarea id="content" name="content" rows="10" required class="mt-1 block w-full px-3 py-2 border border-gray-300 rounded-md shadow-sm focus:outline-none focus:ring-brand-red focus:border-brand-red"></textarea>
                </div>
                <div class="flex items-center space-x-4">
                    <button type="submit" class="px-8 py-3 bg-green-600 text-white font-bold rounded-lg shadow-md hover:bg-green-700 transition duration-300">
                        Publicar Noticia
                    </button>
                    <button type="button" id="logout-button" class="hidden px-6 py-2 bg-gray-600 text-white font-semibold rounded-lg shadow-md hover:bg-gray-700 transition duration-300">
                        Cerrar Sesión
                    </button>
                </div>
            </form>

        </section>

        <!-- Feed de Noticias Nacionales (Con layout de Héroe) -->
        <section id="nacional-section" class="mb-12">
            <h2 class="text-4xl font-bold text-gray-900 border-b-4 border-brand-red pb-3 mb-8">
                Nacional
            </h2>
            <div id="nacional-feed" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                <!-- Las tarjetas de noticias se cargarán aquí -->
                <!-- Placeholder para layout Héroe -->
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse lg:col-span-2 lg:row-span-2">
                    <div class="w-full h-96 bg-gray-300"></div>
                    <div class="p-8">
                        <div class="h-10 bg-gray-300 rounded w-3/4 mb-4"></div>
                        <div class="h-4 bg-gray-300 rounded w-full mb-2"></div>
                        <div class="h-4 bg-gray-300 rounded w-1/2"></div>
                    </div>
                </div>
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div class="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div class="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
            </div>
        </section>

        <!-- Feed de Noticias Internacionales (Grid estándar) -->
        <section id="internacional-section" class="mb-12">
            <h2 class="text-4xl font-bold text-gray-900 border-b-4 border-brand-red pb-3 mb-8">
                Internacional
            </h2>
            <div id="internacional-feed" class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
                <!-- Las tarjetas de noticias se cargarán aquí -->
                <!-- Placeholders para Grid -->
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div class="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div class="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div class="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
                <div class="bg-white rounded-lg shadow-lg overflow-hidden animate-pulse">
                    <div class="w-full h-48 bg-gray-300"></div>
                    <div classD="p-6"><div class="h-6 bg-gray-300 rounded w-3/4 mb-4"></div></div>
                </div>
            </div>
        </section>

    </main>

    <!-- Footer -->
    <footer class="bg-gray-900 text-gray-400 py-12 mt-12">
        <div class="container mx-auto px-4 text-center">
            <h3 class="text-2xl font-bold text-white mb-4">La Movida Noticias</h3>
            <p class="mb-4">El periodismo independiente que te mueve.</p>
            <p class="text-sm">&copy; 2025 La Movida Noticias. Todos los derechos reservados.</p>
        </div>
    </footer>

</body>
</html>

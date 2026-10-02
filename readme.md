LingoDoc System: AI-Powered English Learning & Parsing

🎯 Descripción del Proyecto

LingoDoc es una aplicación web y de procesamiento de archivos diseñada para ayudar en el aprendizaje del inglés. Permite a los usuarios subir documentos (texto plano, Word, PDF) e imágenes (fotos, documentos escaneados, texto escrito a mano).

El sistema extrae el texto, procesa su contenido mediante Modelos de Lenguaje Grandes (LLMs) locales, y genera una salida estructurada que incluye:

El texto original en inglés.

Una aproximación fonética "leída en español".

(Opcional) La traducción al español.

Los resultados se almacenan en archivos .md (Markdown), utilizando un sistema de búsqueda semántica (Embeddings) para sugerir a qué archivo existente se le debe hacer append (agregar el contenido) o si se debe crear uno nuevo. Todo esto disponible a través de un catálogo web.

🛠 Stack Tecnológico

El proyecto utiliza una arquitectura basada en microservicios, optimizada para despliegues on-premise o locales con recursos limitados.

1. Frontend (Interfaz de Usuario)

Framework: Vue 3 (Composition API, Vite).

Funcionalidades: Drag & Drop de archivos, Toggle para traducción opcional, visualizador de Markdown para el catálogo de archivos, e interfaz de selección de coincidencias semánticas.

2. Backend Core (Orquestador y Gestor de Archivos)

Lenguaje: Go (Golang).

Rol principal: Servir la API principal, manejar la concurrencia de subidas, orquestar las llamadas al servicio de IA, interactuar con la base de datos y escribir/modificar los archivos .md físicos en el disco.

3. Backend AI (Procesamiento y Extracción)

Framework: Python + FastAPI.

Rol principal: Detectar el tipo de archivo y enrutarlo (Extracción de texto tradicional vs. Visión Artificial), y comunicarse con el motor LLM.

Librerías: PyPDF2, python-docx (para ruta rápida de texto).

4. Base de Datos Vectorial

Tecnología: PostgreSQL + extensión pgvector.

Rol principal: Almacenar los embeddings (representaciones vectoriales) de los archivos .md existentes para buscar coincidencias semánticas rápidas.

🧠 Motor LLM: Ollama vs. llama.cpp

Para cumplir con el requerimiento de consumir pocos recursos y ser de fácil implementación, se analizan las dos opciones principales:

llama.cpp: Es el motor base en C/C++. Consume la menor cantidad absoluta de recursos (sin procesos en segundo plano). Sin embargo, es difícil de orquestar si necesitas cambiar rápidamente entre un modelo de Visión y uno de Texto, requiriendo scripts complejos para cargar y descargar modelos de la RAM.

Ollama: Utiliza llama.cpp por debajo, por lo que hereda su excelente rendimiento y bajo consumo (soporte para Apple Metal, CUDA, AVX2). Su gran ventaja es que es extremadamente fácil de usar. Ofrece una API REST nativa y gestiona la memoria automáticamente (carga y descarga modelos de la RAM según se necesiten).

✅ Decisión Oficial del Stack: Ollama. Es el equilibrio perfecto. Ofrece el mismo bajo consumo que llama.cpp, pero automatiza el intercambio entre modelos (Visión, Texto y Embeddings), lo cual es crítico para esta arquitectura.

Modelos Abiertos y Ligeros (Cuantizados en GGUF 4-bit)

Modelo de Texto y Formato: qwen2.5:3b (Consume ~3GB RAM). Excelente seguimiento de instrucciones y bilingüismo.

Modelo de Visión (OCR/Mano): llama3.2-vision:11b (Consume ~8GB RAM). Excepcional leyendo escritura a mano.

Modelo de Embeddings: nomic-embed-text (Consume <1GB RAM). Para vectorizar el contenido.

⚙️ Arquitectura de Flujo de Datos (Pipeline)

Para los LLMs o desarrolladores leyendo esto, el flujo de ejecución es el siguiente:

Fase 1: Recepción y Enrutamiento (Go -> Python)

El usuario sube un archivo vía Vue 3 al endpoint de Go.

Go guarda el archivo temporalmente y lo envía al microservicio de Python.

Python evalúa el MIME type:

Ruta Rápida (TXT, DOCX, PDF nativo): Extrae texto con librerías nativas.

Ruta Visual (PNG, JPG, PDF escaneado): Envía la imagen a Ollama (llama3.2-vision) para extraer el texto.

Fase 2: Procesamiento Semántico (Python -> Ollama)

Con el texto plano recuperado, Python hace un llamado a Ollama (qwen2.5:3b) con el siguiente System Prompt Estricto:

"Eres un asistente bilingüe. Tu tarea es extraer o formatear el texto en inglés proporcionado, generar su traducción al español (si se requiere) y crear una aproximación fonética usando las reglas de lectura del idioma español. Formato requerido: ([pronunciación leída en español]) \n [Texto en inglés] | [Traducción al español]"

Python devuelve la estructura de texto final a Go.

Fase 3: Búsqueda Semántica y Guardado (Go -> PostgreSQL)

Go recibe el texto procesado.

Go solicita a Ollama (nomic-embed-text) el embedding matemático del nuevo texto.

Go consulta a PostgreSQL (pgvector) buscando los 3 archivos .md más similares semánticamente.

Se devuelve la información a Vue 3. El usuario elige si hacer append a una coincidencia (ej: notas_clase.md) o crear uno nuevo (nuevo_archivo.md).

Go escribe la información en el disco físico.

📝 Ejemplo de Salida (Output Esperado)

Si el usuario procesa una imagen que dice "I am Carlos and live in Bogota" con el flag de traducción activo, el texto a inyectar en el Markdown será:

(__ia am Carlos an. live bogota___)
I am Carlos and live in bogota | yo soy Carlos y vivo en Bogota


🚀 Requisitos de Infraestructura (Local)

CPU: Procesador con soporte AVX2 (Intel Gen 8+ / AMD Ryzen) o ARM (Apple Silicon M1/M2/M3).

RAM: 16 GB recomendados (8GB dedicados exclusivamente al sistema Ollama para modelos).

Almacenamiento: SSD NVMe obligatorio para la carga rápida (hot-swapping) de los pesos de los modelos.
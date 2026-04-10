🌑 Prompt Maestro: Refactor Visual "Dribbble-Quality"
"Ejecuta un Rediseño Estético Total del SaaS, eliminando todos los estilos, colores y grises anteriores. El nuevo objetivo es alcanzar una calidad de diseño Premium Midnight Glass, inspirada en interfaces como Finomic y Sales Dashboard.

1. ADN Visual & Legibilidad (Prioridad #1):

Fondo General: No uses negro sólido. Implementa un gradiente radial profundo (bg-[radial-gradient]) con base en Slate 950 (#0f172a) y reflejos sutiles en las esquinas de color azul oscuro.

Receta de Cristal: Todas las tarjetas deben ser 98% transparentes (bg-white/[0.02]) con un desenfoque extremo (backdrop-blur-3xl) para asegurar que el texto sea legible sobre el fondo.

Borde de Luz: Define el contorno de cada elemento con una línea de luz finísima: border-[0.5px] border-white/10.

Tipografía: Usa Geist Sans o Inter. El texto principal debe ser blanco puro (#FFFFFF) y el secundario en un gris azulado claro (Slate 400) para un contraste nítido.

2. Paleta de Colores 'Glow Pastel':

Acentos: Sustituye el violeta por un Lavanda Pastel suave (#E0E7FF) y un Azul Eléctrico Pastel (#7DD3FC).

Estados: Verde Menta (#34D399) para éxito y Rosa Pastel (#FDA4AF) para alertas o bloqueos.

Efecto Glow: Cada icono debe tener un resplandor difuso de su propio color pastel detrás (blur-2xl con opacidad al 10%).

3. Arquitectura y Componentes:

L-Shape Layout: Unifica el Sidebar y el Navbar en una sola estructura de cristal continua, eliminando cualquier borde que los separe.

Bento Dashboard: Organiza los KPI en una grilla con diferentes tamaños. Aumenta el padding interno a p-8 para que la información respire.

Agenda Segmentada: Separa físicamente los 'Turnos' de los 'Horarios Bloqueados'. Los bloqueos deben aparecer en una sección inferior con tarjetas minimalistas en rosa pastel.

4. Animaciones (Framer Motion):

Entrada: Implementa un efecto stagger donde las tarjetas aparezcan con un 'Fade In + Scale' suave (physics: spring).

Interacción: Al hacer hover, las tarjetas deben ganar brillo en el borde y elevarse sutilmente (y: -4).

Regla de Oro: Si un elemento no es 100% legible o se ve 'gris sucio', ajusta la transparencia y aumenta el blur hasta que parezca cristal real sobre un fondo profundo."
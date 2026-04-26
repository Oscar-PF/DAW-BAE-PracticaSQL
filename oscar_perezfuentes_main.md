# Gestión de Ocio Personal

## 1. Descripción de requisitos
Esta base de datos está diseñada para gestionar un catálogo personal de consumo cultural (obras). Los requisitos principales son:
* Clasificar las obras por **tipos** (Libros, Series, Películas, etc.).
* Almacenar metadatos de las **obras** como el título, autor y año de publicación.
* Llevar un **registro** del progreso personal, permitiendo marcar estados (Iniciado, Terminado, Abandonado), asignar una fecha de inicio y una nota numérica del 0 al 10.
* Garantizar la integridad referencial: no se puede borrar un tipo si tiene obras asociadas (`RESTRICT`), pero si se elimina una obra, se borran automáticamente sus registros de progreso (`CASCADE`).

## 2. Diagrama del Modelo Relacional
![Diagrama de Base de Datos](./Gestor%20de%20ocio.drawio.svg)

## 3. Script de creación y datos de prueba

```sql
-- 1. Tipos (Series, Libros, Pelis, Juegos...)
CREATE TABLE tipos (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    nombre VARCHAR(255) NOT NULL UNIQUE
);

-- 2. El catálogo de lo que tengo o quiero
CREATE TABLE obras (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    titulo VARCHAR(255) NOT NULL,
    autor VARCHAR(255), -- Autor, Director o Estudio
    publicacion INT,
    tipo_id INT,
    CONSTRAINT fk_tipo 
        FOREIGN KEY (tipo_id) 
        REFERENCES tipos(id) 
        ON DELETE RESTRICT
);

-- 3. Mi progreso personal
CREATE TABLE registros (
    id INT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    estado VARCHAR(255) DEFAULT 'Iniciado',
    iniciado DATE NOT NULL,
    nota INT CHECK (nota >= 0 AND nota <= 10),
    finalizado DATE,
    obra_id INT NOT NULL,
    CONSTRAINT fk_obra 
        FOREIGN KEY (obra_id) 
        REFERENCES obras(id) 
        ON DELETE CASCADE, -- Si borro la obra del catálogo, se borra mi progreso
    CONSTRAINT check_estado 
        CHECK (estado IN ('Iniciado', 'Terminado', 'Abandonado'))
);


-- Datos de prueba 
-- Insertar tipos
INSERT INTO tipos (nombre) VALUES ('Libro'), ('Videojuego'), ('Serie'), ('Pelicula');

-- Insertar contenidos
INSERT INTO obras (titulo, autor, publicacion, tipo_id) 
VALUES ('The Witcher 3', 'CD Projekt Red', 2015, (SELECT id FROM tipos WHERE nombre = 'Videojuego')),
       ('Dune', 'Frank Herbert', 1965, (SELECT id FROM tipos WHERE nombre = 'Libro')),
			 ('Lost', 'ABC', 2004, (SELECT id FROM tipos WHERE nombre = 'Serie')),
			 ('It', 'Warner Bros. Pictures', 2017, (SELECT id FROM tipos WHERE nombre = 'Pelicula')),
			 ('Interstellar', 'Christopher Nolan', 2014, (SELECT id FROM tipos WHERE nombre = 'Pelicula')),
       ('The Last of Us Part II', 'Naughty Dog', 2020, (SELECT id FROM tipos WHERE nombre = 'Videojuego')),
       ('1984', 'George Orwell', 1949, (SELECT id FROM tipos WHERE nombre = 'Libro')),
       ('Breaking Bad', 'Vince Gilligan', 2008, (SELECT id FROM tipos WHERE nombre = 'Serie'));

-- Insertar registros
INSERT INTO registros (estado, iniciado, nota, finalizado, obra_id) 
VALUES ('Terminado', '2015-10-2', 10, '2016-3-25', (SELECT id FROM obras WHERE titulo = 'The Witcher 3')),
       ('Iniciado', '2025-3-15', NULL, NULL, (SELECT id FROM obras WHERE titulo = 'Dune')),
			 ('Terminado', '2024-01-10', 9, '2024-01-10', (SELECT id FROM obras WHERE titulo = 'Interstellar')),
       ('Iniciado', '2025-01-01', NULL, NULL, (SELECT id FROM obras WHERE titulo = 'The Last of Us Part II')),
       ('Terminado', '2023-05-20', 10, '2023-06-10', (SELECT id FROM obras WHERE titulo = '1984')),
       ('Abandonado', '2022-11-12', 3, '2022-12-18', (SELECT id FROM obras WHERE titulo = 'Lost'));
```


## 4. Consultas relevantes

#### Consulta 1: Listado completo de obras con su categoría

```sql
SELECT o.titulo, o.autor, t.nombre AS tipo 
FROM obras o 
JOIN tipos t ON o.tipo_id = t.id 
ORDER BY t.nombre, o.titulo;
```
**Resultado**:
| titulo         |         autor         |    tipo    |
|------------------------+-----------------------+------------ |
| 1984                   | George Orwell         | Libro |
| Dune                   | Frank Herbert         | Libro |
| Interstellar           | Christopher Nolan     | Pelicula |
| It                     | Warner Bros. Pictures | Pelicula |
| Breaking Bad           | Vince Gilligan        | Serie |
| Lost                   | ABC                   | Serie |
| The Last of Us Part II | Naughty Dog           | Videojuego |
| The Witcher 3          | CD Projekt Red        | Videojuego |
(8 rows)


#### Consulta 2: Tiempo empleado en finalizar las obras

```sql
SELECT o.titulo, (finalizado - iniciado) AS dias_empleados 
FROM registros r 
JOIN obras o ON r.obra_id = o.id 
WHERE estado = 'Terminado';
```
**Resultado**:
|titulo     | dias_empleados       |
|---------------+---------------- |
| The Witcher 3 |            175 |
| Interstellar  |             36 |
| 1984          |             21 |
(3 rows)

#### Consulta 3: Ranking de mejores autores por nota media

```sql
SELECT o.autor, ROUND(AVG(r.nota), 2) as nota_media
FROM obras o
JOIN registros r ON o.id = r.obra_id
WHERE r.nota IS NOT NULL
GROUP BY o.autor
ORDER BY nota_media DESC;
```
**Resultado**:
|       autor       | nota_media  |
|-------------------+------------ |
| George Orwell     |      10.00 |
| CD Projekt Red    |      10.00 |
| Christopher Nolan |       9.00 |
| ABC               |       4.00 |
(4 rows)

#### Consulta 4: Cantidad de obras por decada

```sql
SELECT 
    (publicacion / 10) * 10 || 's' AS decada,
    COUNT(*) AS total
FROM obras
WHERE publicacion IS NOT NULL
GROUP BY decada
ORDER BY decada ASC;
```
**Resultado**:
| decada | total  |
|--------+------- |
| 1940s  |     1 |
| 1960s  |     1 |
| 2000s  |     2 |
| 2010s  |     3 |
| 2020s  |     1 |
(5 rows)

#### Consulta 5: Obras que han sido abandonadas

```sql
SELECT o.titulo, o.autor, r.nota 
FROM registros r 
JOIN obras o ON r.obra_id = o.id 
WHERE r.estado = 'Abandonado';
```

**Resultado**:
| titulo | autor | nota  |
|--------+-------+------ |
| Lost   | ABC   |    4 |
(1 row)


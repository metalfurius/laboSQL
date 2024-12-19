-- Eliminación de la base de datos previa si existe
DROP DATABASE IF EXISTS Videojuegos;

-- Creación de la base de datos
CREATE DATABASE Videojuegos;
USE Videojuegos;

-- Eliminación de tablas para reiniciar el esquema
DROP TABLE IF EXISTS Progresos;
DROP TABLE IF EXISTS Jugadores;
DROP TABLE IF EXISTS Videojuegos;
DROP TABLE IF EXISTS Valoraciones;

-- Creación de la tabla Videojuegos
CREATE TABLE Videojuegos(
    videojuegoId INT AUTO_INCREMENT NOT NULL PRIMARY KEY,
    nombre VARCHAR(80) NOT NULL,
    fechaLanzamiento DATE,
    logros INT,
    estado ENUM('Lanzado', 'Beta', 'Acceso anticipado'),
    precioLanzamiento DOUBLE
);

-- Creación de la tabla Jugadores
CREATE TABLE Jugadores(
    jugadorId INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    nickname VARCHAR(60) NOT NULL
);

-- Creación de la tabla Valoraciones
CREATE TABLE Valoraciones(
    usuarioId INT,
    videojuegoId INT,
    fechaValoracion DATE NOT NULL,
    puntuacion DECIMAL(2,1) NOT NULL,
    opinion TEXT NOT NULL,
    numLikes INT DEFAULT 0,
    veredicto ENUM('Imprescindible', 'Recomendado', 'Comprar en rebajas', 'No merece la pena') NOT NULL,
    PRIMARY KEY(usuarioId,videojuegoId),
    FOREIGN KEY(usuarioId) REFERENCES Jugadores(jugadorId),
    FOREIGN KEY(videojuegoId) REFERENCES Videojuegos(videojuegoId),
    CONSTRAINT check_puntuacion CHECK (puntuacion >= 0 AND puntuacion <= 5)
);

-- Inserción de datos en la tabla Videojuegos
INSERT INTO Videojuegos(nombre, fechaLanzamiento, logros, estado, precioLanzamiento) VALUES 
('The Legend of Zelda: Breath of the Wild', '2017-03-03', 76, 'Lanzado', 69.99),
('The Legend of Zelda: Tears of the Kingdom', '2023-05-12', 139, 'Lanzado', 79.99),
('Maniac Mansion', '1987-01-01', 1, 'Lanzado', 49.98),
('Horizon: Zero Dawn', '2017-02-28', 31, 'Lanzado', 79.99),
('Super Metroid', '1994-04-28', 1, 'Lanzado', 69.99),
('Final Fantasy IX', '2001-02-16', 9, 'Lanzado', 69.99),
('Pokemon Rojo', '1999-11-01', 151, 'Lanzado', 49.98),
('Pokemon Amarillo', '2000-06-16', 155, 'Lanzado', 49.98),
('Pokemon Beige Clarito', '2023-12-15', 3, 'Beta', 2000000);

-- Inserción de datos en la tabla Jugadores
INSERT INTO Jugadores(nickname) VALUES ('Currito92'), ('MariTrini67'), ('IISSI_USER'), ('Samus'), ('Aran');

-- Procedimiento para insertar valoraciones
DELIMITER //
CREATE PROCEDURE insertarValoracion(
    IN p_usuarioId INT,
    IN p_videojuegoId INT,
    IN p_fecha DATE,
    IN p_puntuacion DECIMAL(2,1),
    IN p_opinion TEXT,
    IN p_veredicto VARCHAR(50)
)
BEGIN
    INSERT INTO Valoraciones (
        usuarioId,
        videojuegoId,
        fechaValoracion,
        puntuacion,
        opinion,
        veredicto
    ) VALUES (
        p_usuarioId,
        p_videojuegoId,
        p_fecha,
        p_puntuacion,
        p_opinion,
        p_veredicto
    );
END //
DELIMITER ;

-- Llamadas al procedimiento insertarValoracion
CALL insertarValoracion(1, 2, CURDATE(), 5, 'Excelente juego', 'Imprescindible');
CALL insertarValoracion(2, 4, CURDATE(), 3, 'Está bien pero caro', 'Comprar en rebajas');
CALL insertarValoracion(3, 3, CURDATE(), 4, 'Muy buen juego', 'Recomendado');
CALL insertarValoracion(4, 5, CURDATE(), 1, 'No me gustó nada', 'No merece la pena');
CALL insertarValoracion(2, 3, CURDATE(), 4.5, 'Me encantó', 'Imprescindible');

-- Casos que deberían fallar (en rojo en el enunciado):
-- Falla por RN-1-01 (puntuación > 5)
-- CALL insertarValoracion(1, 6, CURDATE(), 10, 'Increíble', 'Imprescindible');

-- Falla por RN-1-02 (veredicto no válido)
-- CALL insertarValoracion(3, 1, CURDATE(), 3, 'Regular', 'Ni fu ni fa');

-- Falla por RN-1-03 (usuario 3 ya valoró juego 3)
-- CALL insertarValoracion(3, 3, CURDATE(), 2, 'No era para tanto', 'No merece la pena');

-- Falla por referencia no existente (usuario 6 y juego 8 no existen)
-- CALL insertarValoracion(6, 8, CURDATE(), 3, 'Buen juego', 'Comprar en rebajas');

-- Cree una consulta que devuelva todos los usuarios, sus juegos y las valoraciones respectivas, ordenados por videojuegosId.

SELECT 
    j.nickname AS NombreUsuario,
    v.nombre AS Videojuego,
    val.puntuacion AS Puntuacion,
    val.veredicto AS Veredicto
FROM Videojuegos v
INNER JOIN Valoraciones val ON v.videojuegoId = val.videojuegoId
INNER JOIN Jugadores j ON j.jugadorId = val.usuarioId
ORDER BY v.videojuegoId;


-- Creación de un trigger para validar fechas de valoración
DELIMITER //
CREATE TRIGGER validar_fecha_valoracion
BEFORE INSERT ON Valoraciones
FOR EACH ROW
BEGIN
    DECLARE fecha_lanzamiento DATE;
    SELECT fechaLanzamiento 
    INTO fecha_lanzamiento
    FROM Videojuegos
    WHERE videojuegoId = NEW.videojuegoId;

    IF NEW.fechaValoracion < fecha_lanzamiento THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'La fecha de valoración no puede ser anterior a la fecha de lanzamiento del videojuego.';
    END IF;

    IF NEW.fechaValoracion > CURDATE() THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'La fecha de valoración no puede ser posterior a la fecha actual.';
    END IF;
END;
//
DELIMITER ;

-- Esta inserción fallará porque la fecha es anterior a la fecha de lanzamiento del juego (2023-05-12)
-- CALL insertarValoracion(1, 2, '2020-01-01', 5, 'Error de fecha', 'Imprescindible');


-- Creación de una función para contar valoraciones por usuario
DELIMITER //
CREATE FUNCTION obtener_numero_valoraciones(p_usuarioId INT)
RETURNS INT
DETERMINISTIC
BEGIN
    DECLARE numero_valoraciones INT;
    SELECT COUNT(*) INTO numero_valoraciones
    FROM Valoraciones
    WHERE usuarioId = p_usuarioId;
    RETURN numero_valoraciones;
END;
//
DELIMITER ;

-- Llamar a la función para UsuarioId=2
SELECT obtener_numero_valoraciones(2) AS NumeroValoraciones;


-- Consulta de videojuegos con su media de valoraciones
SELECT 
    v.nombre AS Juego,
    ROUND(AVG(val.puntuacion), 2) AS MediaValoraciones
FROM 
    Videojuegos v
LEFT JOIN 
    Valoraciones val ON v.videojuegoId = val.videojuegoId
GROUP BY 
    v.videojuegoId
ORDER BY 
    MediaValoraciones DESC;

-- Creación de un trigger para impedir valoraciones en juegos en Beta
DELIMITER //
CREATE TRIGGER impedir_valoracion_juego_beta
BEFORE INSERT ON Valoraciones
FOR EACH ROW
BEGIN
    DECLARE estado_juego VARCHAR(20);
    SELECT estado
    INTO estado_juego
    FROM Videojuegos
    WHERE videojuegoId = NEW.videojuegoId;

    IF estado_juego = 'Beta' THEN
        SIGNAL SQLSTATE '45000'
        SET MESSAGE_TEXT = 'No se puede valorar un juego que esté en fase Beta.';
    END IF;
END;
//
DELIMITER ;

-- Videojuego "Pokemon Beige Clarito" en estado Beta
-- CALL insertarValoracion( 1, 9, CURDATE(), 4.5, 'Es interesante, pero le falta contenido.', --- 'Recomendado');


-- Creación de un procedimiento para insertar un usuario y una valoración en una transacción
DELIMITER //
CREATE PROCEDURE pAddUsuarioValoracion(
    IN p_nickname VARCHAR(60),
    IN p_videojuegoId INT,
    IN p_fecha DATE,
    IN p_puntuacion DECIMAL(2,1),
    IN p_opinion TEXT,
    IN p_veredicto VARCHAR(50)
)
BEGIN
    DECLARE v_usuarioId INT;

    DECLARE EXIT HANDLER FOR SQLEXCEPTION 
    BEGIN
        ROLLBACK;
        SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'Transacción abortada: Error durante la inserción.';
    END;

    START TRANSACTION;
    INSERT INTO Jugadores(nickname) VALUES (p_nickname);
    SET v_usuarioId = LAST_INSERT_ID();
    CALL insertarValoracion(v_usuarioId, p_videojuegoId, p_fecha, p_puntuacion, p_opinion, p_veredicto);
    COMMIT;
END;
//
DELIMITER ;

-- Ejemplo de uso del procedimiento pAddUsuarioValoracion
CALL pAddUsuarioValoracion('NuevoUsuarioLocal', 8, CURDATE(), 4.5, 'Es interesante, pero le falta contenido.', 'Recomendado');
-- Ejemplo de caso fallido
-- CALL pAddUsuarioValoracion('UsuarioFallidoLocal', 9, CURDATE(), 4.5, 'Intento fallido porque el juego está en Beta.', 'Recomendado');

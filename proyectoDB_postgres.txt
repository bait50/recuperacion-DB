--
CREATE TABLE IF NOT EXISTS public.categorias
(
    idcategoria SERIAL PRIMARY KEY, -- Generación automática de ID
    nombre CHARACTER VARYING(100) NOT NULL,
    estado CHARACTER(1) NOT NULL -- Columna para el estado ('A', 'I', etc.)
);

-- Inserción de datos en categorias
INSERT INTO public.categorias ( nombre, estado)
VALUES
    ( 'Lácteos', 'A'),
    ( 'Granos', 'A'),
    ( 'Harinas', 'A'),
    ( 'Limpieza','A');

--
CREATE SEQUENCE IF NOT EXISTS productos_idProd_seq;
--
CREATE TABLE IF NOT EXISTS public.productos
(
    "idProd" INTEGER NOT NULL DEFAULT nextval('productos_idProd_seq'::regclass),
    nombre CHARACTER VARYING(200) NOT NULL,
    stock NUMERIC(5, 0) NOT NULL,
    precio NUMERIC(10, 2) NOT NULL,
    estado CHARACTER(1) NOT NULL,
    idcategoria INTEGER,
    "valorTotal" NUMERIC(10, 2) NOT NULL,
    CONSTRAINT pk_productos PRIMARY KEY ("idProd"),
    CONSTRAINT fk_idcategoria FOREIGN KEY (idcategoria)
        REFERENCES public.categorias (idcategoria)
        ON UPDATE NO ACTION
        ON DELETE NO ACTION
);

INSERT INTO public.productos (nombre, stock, precio, estado, idcategoria, "valorTotal")
VALUES
    ('Jabón Ariel', 5, 34000.00, 'A', 4, 170000.00),
    ('Escoba', 7, 14900.00, 'A', 4, 104300.00),
    ('Leche', 20, 5390.00, 'A', 1, 98000.00),
    ('Lenteja', 10, 5057.80, 'A', 2, 38000.00),
    ('Frijol', 20, 12644.50, 'A', 2, 190000.00);

--

    CREATE TABLE IF NOT EXISTS public.auditoria_productos
(  id SERIAL Primary key,
   idProd INT NOT NULL,  -- guardar el id del usuario que se afecto
   operacion VARCHAR(10) NOT NULL, -- UPDATE, DELETE, INSERT
   fecha TIMESTAMP DEFAULT now(),
   datos JSONB
);

--
CREATE OR REPLACE FUNCTION auditar_productos()
RETURNS TRIGGER
AS
$$
BEGIN
	IF(OLD.precio IS DISTINCT FROM NEW.precio) THEN
		INSERT INTO public.auditoria_productos (idProd, operacion, datos)
		VALUES (NEW."idProd",'UPDATE', row_to_json('old',row_to_json(OLD.precio),'new',row_to_json(NEW.precio)));
	ELSIF(OLD.stock IS DISTINCT FROM NEW.stock) THEN
		INSERT INTO public.auditoria_productos (idProd, operacion, datos)
		VALUES (NEW."idProd",'UPDATE', row_to_json('old',row_to_json(OLD.stock),'new',row_to_json(NEW.stock)));
	END IF;
	RETURN NULL;
END;
$$
LANGUAGE plpgsql;


CREATE TRIGGER trigger_auditar_productos
AFTER UPDATE ON public.productos
FOR EACH ROW
EXECUTE FUNCTION auditar_productos();


--
CREATE TABLE IF NOT EXISTS public.auditoria_cambios_productos
(  id SERIAL Primary key,
   idProd INT NOT NULL,  -- guardar el id del usuario que se afecto
   operacion VARCHAR(10) NOT NULL, -- UPDATE, DELETE, INSERT
   fecha TIMESTAMP DEFAULT now(),
   datos JSONB
);

CREATE OR REPLACE FUNCTION auditoria_cambios_productos()
RETURNS TRIGGER
AS
$$
BEGIN
	IF (TG_OP = 'DELETE')THEN
		INSERT INTO public.auditoria_cambios_productos (idProd, operacion, datos) VALUES (OLD."idProd",'DELETE', row_to_json(OLD));

	ELSEIF (TG_OP = 'UPDATE') THEN
	    INSERT INTO public.auditoria_cambios_productos (idProd, operacion, datos)
		VALUES (NEW."idProd",'UPDATE', row_to_json('old',row_to_json(OLD),'new',row_to_json(NEW)));
	ELSEIF (TG_OP = 'INSERT') THEN
		 INSERT INTO public.auditoria_cambios_productos (idProd, operacion, datos) VALUES (NEW."idProd",'INSERT', row_to_json(NEW));
	END IF;
	RETURN NULL;
END;
$$
LANGUAGE plpgsql;


CREATE TRIGGER trigger_auditoria_cambios_productos
AFTER INSERT OR UPDATE OR DELETE
ON productos
FOR EACH ROW
EXECUTE FUNCTION auditoria_cambios_productos();

--funciones
--insertar_producto
CREATE OR REPLACE FUNCTION insertar_producto(
    _nombre CHARACTER VARYING(200),
    _stock NUMERIC(5, 0),
    _precio NUMERIC(10, 2),
    _estado CHARACTER(1),
    _idcategoria INTEGER,
    _valorTotal NUMERIC(10, 2)
)
RETURNS VOID AS $$
BEGIN
    INSERT INTO public.productos (nombre, stock, precio, estado, idcategoria, "valorTotal")
    VALUES (_nombre, _stock, _precio, _estado, _idcategoria, _valorTotal);
END;
$$ LANGUAGE plpgsql;

SELECT setval('productos_idProd_seq', (SELECT MAX("idProd") FROM productos)); --Esto ajusta la secuencia productos_idProd_seq al valor máximo actual de idProd en la tabla productos.


SELECT insertar_producto('Limpiador Multiuso', 15, 12000.00, 'A', 4, 180000.00);
SELECT insertar_producto('Desinfectante Líquido', 8, 7500.00, 'A', 4, 60000.00);
SELECT insertar_producto('Yogur de Fresa', 30, 3500.00, 'A', 1, 105000.00);
SELECT insertar_producto('Queso Mozzarella', 20, 8200.00, 'A', 1, 164000.00);
SELECT insertar_producto('Arroz Integral', 25, 4200.00, 'A', 2, 105000.00);
SELECT insertar_producto('Maíz Amarillo', 18, 3650.00, 'A', 2, 65700.00);
SELECT insertar_producto('Harina de Trigo', 50, 2500.00, 'A', 3, 125000.00);
SELECT insertar_producto('Harina de Maíz', 40, 3100.00, 'A', 3, 124000.00);
SELECT insertar_producto('Detergente en Polvo', 10, 29000.00, 'A', 4, 290000.00);
SELECT insertar_producto('Crema de Leche', 12, 5000.00, 'A', 1, 60000.00);
SELECT insertar_producto('Avena en Hojuelas', 35, 3700.00, 'A', 2, 129500.00);
SELECT insertar_producto('Tortillas de Harina', 15, 2800.00, 'A', 3, 42000.00);

select * from public.productos;

--consultar_producto
CREATE OR REPLACE FUNCTION consultar_producto(_idProd INTEGER)
RETURNS TABLE (
    "idProd" INTEGER,
    nombre CHARACTER VARYING(200),
    stock NUMERIC(5, 0),
    precio NUMERIC(10, 2),
    estado CHARACTER(1),
    idcategoria INTEGER,
    "valorTotal" NUMERIC(10, 2)
) AS $$
BEGIN
    RETURN QUERY
    SELECT p."idProd", p.nombre, p.stock, p.precio, p.estado, p.idcategoria, p."valorTotal"
    FROM public.productos p
    WHERE p."idProd" = _idProd;
END;
$$ LANGUAGE plpgsql;


SELECT * FROM consultar_producto(1);

--eliminar_producto
CREATE OR REPLACE FUNCTION eliminar_producto(_idProd INTEGER)
RETURNS VOID AS $$
BEGIN
    DELETE FROM public.productos
    WHERE "idProd" = _idProd;
END;
$$ LANGUAGE plpgsql;

SELECT eliminar_producto(1);

select * from public.productos p ;

--editar_producto
CREATE OR REPLACE FUNCTION editar_producto(
    _idProd INTEGER,
    _nombre CHARACTER VARYING(200),
    _stock NUMERIC(5, 0),
    _precio NUMERIC(10, 2),
    _estado CHARACTER(1),
    _idcategoria INTEGER,
    _valorTotal NUMERIC(10, 2)
)
RETURNS VOID AS $$
BEGIN
    UPDATE public.productos
    SET nombre = _nombre,
        stock = _stock,
        precio = _precio,
        estado = _estado,
        idcategoria = _idcategoria,
        "valorTotal" = _valorTotal
    WHERE "idProd" = _idProd;
END;
$$ LANGUAGE plpgsql;

SELECT editar_producto(1, 'Jabón Ariel Mejorado', 200, 34000.00, 'A', 4, 6800000.00);

select * from public.productos p ;






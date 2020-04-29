# disparadore-en-postgresql
primer trabajo del segundo corte 

create table persona(
per_cc int8 not null,
per_nombre varchar(50) not null,
per_apellido varchar(60) not null,
per_sexo char(1),
per_fechanaci date,
per_edad int2,
per_direccion varchar(80) not null,
per_telefono int8,
primary key (per_cc));
create table guia(
gui_id int8 not null,
gui_fechaEnvio date not null,
gui_fechaLlegada date not null,
gui_duracionEnvio int2 not null,
gui_peso varchar(20) not null,
gui_valor float8 not null,
gui_contenido varchar(200) not null, 
primary key (gui_id),
foreign key (gui_contenido) references contenido(con_id));
create table PERXGUI(
PXG_id int4 not null,
PXG_percc int8 not null,
PXG_guiid int8 not null,
PXG_tipo varchar(50) not null,
primary key (PXG_id),
foreign key (PXG_percc) references persona(per_cc),
foreign key (PXG_guiid) references guia(gui_id));
--------------------------------------------------
---auditoria 
CREATE schema audit;
REVOKE CREATE ON schema audit FROM public;
 
CREATE TABLE audit.logged_actions (
    schema_name text NOT NULL,
    TABLE_NAME text NOT NULL,
    user_name text,
    action_tstamp TIMESTAMP WITH TIME zone NOT NULL DEFAULT CURRENT_TIMESTAMP,
    action TEXT NOT NULL CHECK (action IN ('I','D','U')),
    original_data text,
    new_data text,
    query text
) WITH (fillfactor=100);
 
REVOKE ALL ON audit.logged_actions FROM public;

GRANT SELECT ON audit.logged_actions TO public;
 
CREATE INDEX logged_actions_schema_table_idx 
ON audit.logged_actions(((schema_name||'.'||TABLE_NAME)::TEXT));
 
CREATE INDEX logged_actions_action_tstamp_idx 
ON audit.logged_actions(action_tstamp);
 
CREATE INDEX logged_actions_action_idx 
ON audit.logged_actions(action);

CREATE OR REPLACE FUNCTION audit.if_modified_func() RETURNS TRIGGER AS $body$
DECLARE
    v_old_data TEXT;
    v_new_data TEXT;
BEGIN
     IF (TG_OP = 'UPDATE') THEN
        v_old_data := ROW(OLD.*);
        v_new_data := ROW(NEW.*);
        INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,original_data,new_data,query) 
        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_old_data,v_new_data, current_query());
        RETURN NEW;
    ELSIF (TG_OP = 'DELETE') THEN
        v_old_data := ROW(OLD.*);
        INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,original_data,query)
        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_old_data, current_query());
        RETURN OLD;
    ELSIF (TG_OP = 'INSERT') THEN
        v_new_data := ROW(NEW.*);
        INSERT INTO audit.logged_actions (schema_name,table_name,user_name,action,new_data,query)
        VALUES (TG_TABLE_SCHEMA::TEXT,TG_TABLE_NAME::TEXT,session_user::TEXT,substring(TG_OP,1,1),v_new_data, current_query());
        RETURN NEW;
    ELSE
        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - Other action occurred: %, at %',TG_OP,now();
        RETURN NULL;
    END IF;
 
EXCEPTION
    WHEN data_exception THEN
        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [DATA EXCEPTION] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
    WHEN unique_violation THEN
        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [UNIQUE] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
    WHEN OTHERS THEN
        RAISE WARNING '[AUDIT.IF_MODIFIED_FUNC] - UDF ERROR [OTHER] - SQLSTATE: %, SQLERRM: %',SQLSTATE,SQLERRM;
        RETURN NULL;
END;
$body$
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = pg_catalog, audit;
--------------------------------------------------
---------------------------------INSERT ----------------------------------
---insert para guia
CREATE OR REPLACE FUNCTION anguia() RETURNS trigger AS
$$
    begin
         if(new.gui_id is null) then 
            raise exception 'el numero de la guia debe ser obligatoria';
         end if;
         if(exists (select* from guia where gui_id=new.gui_id) ) then 
            raise exception 'el numeronde guia ya existe';
         end if; 
         if(new.gui_fechaEnvio > new.gui_fechaLlegada) then 
            raise exception 'la fecha de envio debe ser menor a la fecha de llegada';
         end if;
         if(new.gui_peso='') then 
            raise exception 'el peso del paquete debe ser obligatoria';
         end if;
         if(new.gui_valor is null) then 
            raise exception 'el valor del paquete debe ser obligatoria';
         end if;
         if(new.gui_contenido='') then 
            raise exception 'el contenido del paquete debe ser obligatoria';
         end if;
         return new;
    end;
$$
  language 'plpgsql';
  create trigger anguia before insert on guia for each row execute procedure anguia(); 

---insert para PERXGUI
CREATE OR REPLACE FUNCTION in_personaporguia() RETURNS trigger AS $$
begin 
       if(new.PXG_id is null) then 
            raise exception 'el numero de la guiaporpesona debe ser obligatoria';
       end if;
       if(exists (select* from PERXGUI where PXG_id=new.PXG_id) ) then 
            raise exception 'el numeronde de guiaporpesona ya existe';
       end if; 
       if(new.PXG_percc is null) then 
            raise exception 'el numero del codigo de la perosna debe ser obligatoria';
       end if;
       if(new.PXG_guiid is null) then 
            raise exception 'el numero de la guia debe ser obligatoria';
       end if;
       if(exists (select* from PERXGUI where PXG_guiid=new.PXG_guiid) ) then 
            raise exception 'el numeronde guia ya existe';
       end if;
       if(new.PXG_tipo='') then 
            raise exception 'el contenido del tipo debe ser obligatoria';
       end if;
       if(not ((new.PXG_tipo='DESTINATARIO') or (new.PXG_tipo='REMITENTE'))) then 
            raise exception 'El tipo debe ser DESTINATARIO o REMITENTE';
       end if;
       if(not exists(select* from persona where per_cc=new.PXG_percc ))then
          raise exception 'el tipo de datos no esta en la base de datos';
       end if;
       if(not exists(select* from guia where gui_id=new.PXG_guiid ))then
            raise exception 'el tipo de datos no esta en la base de datos';
         end if; 
       return new;
    end;
$$
  language 'plpgsql';
  create trigger in_personaporguia before insert on PERXGUI for each row execute procedure in_personaporguia();

  ---insert para persona 
CREATE OR REPLACE FUNCTION in_persona() RETURNS trigger AS $$
begin 
       if(new.per_cc is null) then 
            raise exception 'el codigo de la persona debe ser obligatoria';
       end if;
       if(exists (select* from persona where per_cc=new.per_cc) ) then 
            raise exception 'el codigo de la persona ya existe';
       end if; 
       if(new.per_nombre='') then 
            raise exception 'el nombre de la perosna debe ser obligatoria';
       end if;
       if(new.per_apellido='') then 
            raise exception 'el apellido de la persona debe ser obligatoria';
       end if;
       if(new.per_direccion='') then 
            raise exception 'la direccion de la persona debe ser obligatoria';
       end if;       
       if(not exists(select* from persona where per_edad >= 18))then 
          raise exception 'la edad de la perosna debe ser mayor de 18 años ';
       end if;
       if(not exists(select* from persona where (new.per_sexo = 'M') or (new.per_sexo='F')))then 
          raise exception 'el sexo de la persona deber ser M para masculino y F para femenino';
       end if;
       return new;
    end;
$$
  language 	
  create trigger in_persona before insert on persona for each row execute procedure in_persona();

  ---------------------------------UPDATE----------------------------------
 ---update para guia 
CREATE OR REPLACE FUNCTION up_anguia() RETURNS trigger AS $$
begin 
       if(new.gui_peso = '50 kg') then
          new.gui_valor = 100000;
          new.gui_contenido = 'carga pesada'; 
       end if;
       if(new.gui_peso= '30 kg') then
          new.gui_valor = 80000;
          new.gui_contenido = 'carga media'; 
       end if;
       if(new.gui_peso = '10 kg') then
          new.gui_valor = 20000;
          new.fui_contenido = 'carga liviana'; 
       end if;
       if(new.gui_contenido = 'fragil') then
          new.gui_valor = old.gui_valor+15000; 
       end if;
       return new;
  end;
$$
  language 'plpgsql';
  create trigger up_anguia before update on guia for each row execute procedure up_anguia();

---before update para PERXGUI
CREATE OR REPLACE FUNCTION up_personaporguia() RETURNS trigger AS $$
begin 
       if(not ((new.PXG_tipo='DESTINATARIO') or (new.PXG_tipo='REMITENTE'))) then 
            raise exception 'El tipo debe ser DESTINATARIO o REMITENTE';
       end if;
       if(not exists(select* from persona where per_cc=new.PXG_percc ))then
          raise exception 'el tipo de datos no esta en la base de datos';
       end if; 
       if(not exists(select* from guia where gui_id=new.PXG_guiid ))then
          raise exception 'el tipo de datos no esta en la base de datos';
       end if;
       return new;
  end;
$$
  language 'plpgsql';
  create trigger up_personaporguia before update on PERXGUI for each row execute procedure up_personaporguia(); 

---before update de persona
CREATE OR REPLACE FUNCTION up_persona() RETURNS trigger AS $$
begin 
       if(not exists(select* from persona where (new.per_sexo = 'M') or (new.per_sexo='F')))then 
          raise exception 'el sexo de la persona deber ser M para masculino y F para femenino';
       end if;
       if(not exists(select* from persona where new.per_edad >= 18))then 
          raise exception 'la edad de la perosna debe ser mayor de 18 años ';
       end if;
       return new;
  end;
$$
  language 'plpgsql';
  create trigger up_persona before update on persona for each row execute procedure up_persona();

---------------------------------DELETE---------------------------------
---before delete de guia
create or replace function de_anguia() returns trigger as $$
begin 
     delete from guia where gui_id = new.gui_id;
     return new;
end;
$$
  language 'plpgsql';
  create trigger de_anguia before delete on guia for each row execute procedure de_anguia();

---before delete de PERXGUI
create or replace function de_personaporguia() returns trigger as $$
begin 
     delete from guia where gui_id = new.gui_id;
     return new;
end;
$$
  language 'plpgsql';
  create trigger de_personaporguia before delete on PERXGUI for each row execute procedure de_personaporguia();

---before delete de persona
create or replace function de_persona() returns trigger as $$
begin 
     delete from guia where gui_id = new.gui_id;
     return new;
end;
$$
  language 'plpgsql';
  create trigger de_persona before delete on persona for each row execute procedure de_persona();

# [Joins with Spring Data R2DBC](https://neilwhite.ca/joins-with-spring-data-r2dbc/)

- [Repositorio del tutorial](https://github.com/neil-writes-code/reactive-spring-demo/tree/main)

---

`Spring Data R2DBC` nos permite escribir código sin bloqueo para interactuar con las bases de datos. A diferencia de los
otros proyectos de `Spring Data`, `Spring Data R2DBC` no es un ORM y tiene algunas limitaciones. **Una de esas
limitaciones es la asignación de combinaciones para entidades.** Esto presenta un desafío para aquellos familiarizados
con `Spring Data JPA`.

> **¿Cómo se escribe código que no bloquee, al mismo tiempo que se unen y mapean entidades complejas?**

## Spring Data R2DBC

Al igual que otros proyectos de `Spring Data`, el objetivo de `Spring Data R2DBC` es facilitar el trabajo con bases de
datos. Para lograrlo de forma reactiva, tuvieron que eliminar muchas funciones. `Spring Data R2DBC` no es un marco ORM
y `no admite joins`.

Para una entidad simple, sin relaciones, `R2DBC` funciona muy bien. Cree una clase de entidad y un `repository` de
soporte.

Cuando se trata de entidades con relaciones, este patrón ya no funciona. Para superar esto, necesitamos crear nuestra
propia implementación de `repository` utilizando `DatabaseClient`.

## Dependencias

````xml
<!--Spring Boo 3.3.3-->
<!--Java 21-->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-r2dbc</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>r2dbc-postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>io.projectreactor</groupId>
        <artifactId>reactor-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
````

## Las entidades

Utilizaremos dos entidades, un `Employee` básico y un `Department` que tiene dos relaciones diferentes con el
`Employee`.

Si traducimos estas entidades a tablas relacionadas en la base de datos, tendríamos el siguiente esquema.

![01.png](assets/01.png)

Para nuestro caso de negocio, diremos lo siguiente:

- Un departamento solo tiene un gerente (manager).
- Un empleado puede ser gerente de un único departamento.
- Un departamento tiene muchos empleados.
- Un empleado solo pertenece a un departamento.

Basándonos en el esquema definido, vamos a crear las instrucciones `DDL` para crear las tablas. Estas instrucciones las
crearemos en el archivo `src/main/resources/schema.sql`.

````sql
CREATE TABLE IF NOT EXISTS employees(
    id BIGSERIAL PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    position VARCHAR(255) NOT NULL,
    is_full_time BOOLEAN NOT NULL
);

CREATE TABLE IF NOT EXISTS departments(
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE IF NOT EXISTS department_managers(
    department_id BIGINT,
    employee_id BIGINT,
    CONSTRAINT pk_dm PRIMARY KEY(department_id, employee_id),
    CONSTRAINT fk_departments_dm FOREIGN KEY(department_id) REFERENCES departments(id),
    CONSTRAINT fk_employees_dm FOREIGN KEY(employee_id) REFERENCES employees(id),
    CONSTRAINT uk_department_id_dm UNIQUE(department_id),
    CONSTRAINT uk_employee_id_dm UNIQUE(employee_id)
);

CREATE TABLE IF NOT EXISTS department_employees(
    department_id BIGINT,
    employee_id BIGINT,
    CONSTRAINT pk_de PRIMARY KEY(department_id, employee_id),
    CONSTRAINT fk_departments_de FOREIGN KEY(department_id) REFERENCES departments(id),
    CONSTRAINT fk_employees_de FOREIGN KEY(employee_id) REFERENCES employees(id),
    CONSTRAINT uk_employee_id_de UNIQUE(employee_id)
);
````

Ahora, en el mismo directorio crearemos el archivo `src/main/resources/data.sql` con las instrucciones `DML` para
poblar las tablas.

Es importante notar que las dos primeras líneas, son instrucciones que permiten truncar las tablas principales
`departments` y `employees`, dado que, cada vez que levantemos la aplicación, vamos a volver a poblar las tablas
con los mismos registros. Usar `CASCADE` eliminará todos los datos de las tablas relacionadas.

````sql
TRUNCATE TABLE departments RESTART IDENTITY CASCADE;
TRUNCATE TABLE employees RESTART IDENTITY CASCADE;

INSERT INTO employees(first_name, last_name, position, is_full_time)
VALUES('Carlos', 'Gómez', 'Gerente', true),
('Ana', 'Martínez', 'Desarrollador', true),
('Luis', 'Fernández', 'Diseñador', false),
('María', 'Rodríguez', 'Analista', true),
('José', 'Pérez', 'Soporte', true),
('Laura', 'Sánchez', 'Desarrollador', true),
('Jorge', 'López', 'Analista', false),
('Sofía', 'Díaz', 'Gerente', true),
('Manuel', 'Torres', 'Soporte', true),
('Lucía', 'Morales', 'Diseñador', true),
('Miguel', 'Hernández', 'Desarrollador', true),
('Elena', 'Ruiz', 'Analista', false),
('Pablo', 'Jiménez', 'Desarrollador', true),
('Carmen', 'Navarro', 'Soporte', true),
('Raúl', 'Domínguez', 'Gerente', true),
('Beatriz', 'Vargas', 'Desarrollador', true),
('Francisco', 'Muñoz', 'Soporte', true),
('Marta', 'Ortega', 'Diseñador', false),
('Andrés', 'Castillo', 'Analista', true),
('Isabel', 'Ramos', 'Desarrollador', true);

INSERT INTO departments(name)
VALUES('Recursos Humanos'),
('Tecnología'),
('Finanzas'),
('Marketing'),
('Ventas');

INSERT INTO department_managers(department_id, employee_id)
VALUES(1, 1),  -- Recursos Humanos - Carlos Gómez
(2, 8),  -- Tecnología - Sofía Díaz
(3, 15), -- Finanzas - Raúl Domínguez
(4, 4),  -- Marketing - María Rodríguez
(5, 20); -- Ventas - Isabel Ramos

INSERT INTO department_employees(department_id, employee_id)
VALUES(1, 5),  -- Recursos Humanos - José Pérez
(1, 7),  -- Recursos Humanos - Jorge López
(2, 2),  -- Tecnología - Ana Martínez
(2, 6),  -- Tecnología - Laura Sánchez
(2, 11), -- Tecnología - Miguel Hernández
(2, 13), -- Tecnología - Pablo Jiménez
(2, 20), -- Tecnología - Isabel Ramos
(3, 10), -- Finanzas - Lucía Morales
(4, 18), -- Marketing - Marta Ortega
(4, 12), -- Marketing - Elena Ruiz
(5, 3),  -- Ventas - Luis Fernández
(5, 9),  -- Ventas - Manuel Torres
(5, 14), -- Ventas - Carmen Navarro
(5, 16), -- Ventas - Beatriz Vargas
(5, 17), -- Ventas - Francisco Muñoz
(5, 19); -- Ventas - Andrés Castillo
````

## Configura propiedades de la aplicación

En el `application.yml` agregaremos las siguientes configuraciones. Notar que la base de datos que estamos usando
se llama `db_webflux_r2dbc`. Esta base de datos la crearemos en el siguiente apartado.

````yml
server:
  port: 8080
  error:
    include-message: always

spring:
  application:
    name: webflux-r2dbc-joins
  r2dbc:
    url: r2dbc:postgresql://localhost:5433/db_webflux_r2dbc
    username: magadiflo
    password: magadiflo

logging:
  level:
    dev.magadiflo.app: DEBUG
    io.r2dbc.postgresql.QUERY: DEBUG
    io.r2dbc.postgresql.PARAM: DEBUG
````

## Base de datos Postgres con docker compose

Vamos a crear nuestra base de datos usando `compose` de docker.

````yml
services:
  postgres:
    image: postgres:15.2-alpine
    container_name: c-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: db_webflux_r2dbc
      POSTGRES_USER: magadiflo
      POSTGRES_PASSWORD: magadiflo
    ports:
      - '5433:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
    name: postgres_data
````

## Crear esquema de base de datos y poblar tablas

Cada vez que iniciemos la aplicación, se ejecutarán los scripts que estamos definiendo en este `@Bean` de configuración,
de esta manera nos aseguramos de que las tablas de la base de dato siempre estén pobladas al iniciar la aplicación.
Cabe resaltar que aunque el archivo `schema.sql` se ejecute cada vez que se inicie la aplicación, solo se crearán las
tablas una sola vez, dado que colocamos en las instrucciónes `DML` lo siguiente `CREATE TABLE IF NOT EXISTS...`.

````java

@Configuration
public class DatabaseConfig {
    @Bean
    public ConnectionFactoryInitializer initializer(ConnectionFactory connectionFactory) {
        ClassPathResource schemaResource = new ClassPathResource("schema.sql");
        ClassPathResource dataResource = new ClassPathResource("data.sql");
        ResourceDatabasePopulator resourceDatabasePopulator = new ResourceDatabasePopulator(schemaResource, dataResource);

        ConnectionFactoryInitializer initializer = new ConnectionFactoryInitializer();
        initializer.setConnectionFactory(connectionFactory);
        initializer.setDatabasePopulator(resourceDatabasePopulator);
        return initializer;
    }
}
````

## Entidades

A continuación se muestran las dos entidades con las que trabajaremos en este proyecto.

````java

@ToString
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Setter
@Getter
@Table(name = "employees")
public class Employee {
    @Id
    private Long id;
    private String firstName;
    private String lastName;
    private String position;
    @Column("is_full_time")
    private boolean fullTime;

    public static Employee fromRow(Map<String, Object> row) {
        if (row.get("e_id") == null) return null;

        return Employee.builder()
                .id(Long.parseLong(row.get("e_id").toString()))
                .firstName((String) row.get("e_firstName"))
                .lastName((String) row.get("e_lastName"))
                .position((String) row.get("e_position"))
                .fullTime((Boolean) row.get("e_isFullTime"))
                .build();
    }

    public static Employee managerFromRow(Map<String, Object> row) {
        if (row.get("m_id") == null) return null;

        return Employee.builder()
                .id(Long.parseLong(row.get("m_id").toString()))
                .firstName((String) row.get("m_firstName"))
                .lastName((String) row.get("m_lastName"))
                .position((String) row.get("m_position"))
                .fullTime((Boolean) row.get("m_isFullTime"))
                .build();
    }
}
````

````java

@ToString
@AllArgsConstructor
@NoArgsConstructor
@Builder
@Setter
@Getter
@Table(name = "departments")
public class Department {
    @Id
    private Long id;
    private String name;

    private Employee manager;

    @Builder.Default
    private List<Employee> employees = new ArrayList<>();

    public Optional<Employee> getManager() {
        return Optional.ofNullable(this.manager);
    }

    public static Mono<Department> fromRows(List<Map<String, Object>> rows) {
        Map<String, Object> firstRow = rows.getFirst();

        Department department = Department.builder()
                .id(Long.parseLong(firstRow.get("d_id").toString()))
                .name(String.valueOf(firstRow.get("d_name")))
                .manager(Employee.managerFromRow(firstRow))
                .employees(rows.stream()
                        .map(Employee::fromRow)
                        .filter(Objects::nonNull)
                        .toList())
                .build();
        return Mono.just(department);
    }

}
````

`@Builder.Default`, esta anotación se usa junto con la anotación `@Builder` de `Lombok`. Cuando utilizas `@Builder` para
generar un patrón de construcción (builder pattern), las variables de instancia que no son explícitamente establecidas
en el proceso de construcción pueden quedar con valores null. Sin embargo, al usar `@Builder.Default`, le indicas a
Lombok que use el valor por defecto proporcionado si no se establece un valor durante la construcción.

Notarás que estas entidades tienen métodos estáticos para crear objetos a partir de una entrada. Estos son los
resultados sin procesar de una llamada con DatabaseClient. Dado que `Spring Data R2DBC` no asigna estos objetos, tenemos
que escribir la lógica nosotros mismos. `row.get` es nuestro mejor amigo aquí, ya que nos permite extraer cualquier
columna y convertirla al tipo que necesitamos.

## Repositorios

Antes dije que los repositorios estándar son adecuados para entidades sin relaciones. Lo que significa que todo lo que
tenemos que hacer para `Employee` es crear nuestra interfaz que extienda de `R2dbcRepository`. Además, estamos creando
consultas personalizadas usando las convenciones de nombre de método.

````java
public interface EmployeeRepository extends R2dbcRepository<Employee, Long> {
    Flux<Employee> findAllByPosition(String position);

    Flux<Employee> findAllByFullTime(boolean isFullTime);

    Flux<Employee> findAllByPositionAndFullTime(String position, boolean isFullTime);

    Mono<Employee> findByFirstName(String firstName);
}
````

Ahora, si revisamos la entidad `Department`, esta entidad está relacionada con `Employee` a través de los atributos
`manager` y `employees`, por lo que crear una interfaz y extenderla de `R2dbcRepository` no es lo adecuado para traer
los datos relacionados.

Lo que haremos en este caso, será crear una interfaz y luego crearemos su propia implementación. Para mantener el mismo
patrón para el `Department`, necesitamos una interfaz y una clase de implementación, `DepartmentRepository` y
`DepartmentRepositoryImpl`.

````java
public interface DepartmentRepository {
    Flux<Department> findAll();

    Mono<Department> findById(Long departmentId);

    Mono<Department> findByName(String name);

    Mono<Department> save(Department department);

    Mono<Void> delete(Department department);
}
````

Damos a cada columna un alias para utilizar al extraer datos para crear nuestros objetos, como se ve en la siguiente
clase de entidad. `No es necesario utilizar alias`, **pero puede resultar útil si el nombre de una columna no coincide
con el nombre del campo.**

````java

@Slf4j
@RequiredArgsConstructor
@Repository
public class DepartmentRepositoryImpl implements DepartmentRepository {

    private final EmployeeRepository employeeRepository;
    private final DatabaseClient client;
    private static final String SELECT_QUERY = """
            SELECT d.id AS d_id,
                    d.name AS d_name,
                    m.id AS m_id,
                    m.first_name AS m_firstName,
                    m.last_name AS m_lastName,
                    m.position AS m_position,
                    m.is_full_time AS m_isFullTime,
                    e.id AS e_id,
                    e.first_name AS e_firstName,
                    e.last_name AS e_lastName,
                    e.position AS e_position,
                    e.is_full_time AS e_isFullTime
            FROM departments AS d
                LEFT JOIN department_managers AS dm ON(d.id = dm.department_id)
                LEFT JOIN employees AS m ON(dm.employee_id = m.id)
                LEFT JOIN department_employees AS de ON(d.id = de.department_id)
                LEFT JOIN employees AS e ON(de.employee_id = e.id)
            """;

    @Override
    public Flux<Department> findAll() {
        return this.client.sql(SELECT_QUERY)
                .fetch()
                .all()
                .bufferUntilChanged(result -> result.get("d_id"))
                .flatMap(Department::fromRows);
    }

    @Override
    public Mono<Department> findById(Long departmentId) {
        return this.client.sql("%s WHERE d.id = :departmentId".formatted(SELECT_QUERY))
                .bind("departmentId", departmentId)
                .fetch()
                .all()
                .bufferUntilChanged(result -> result.get("d_id"))
                .flatMap(Department::fromRows)
                .singleOrEmpty();
    }

    @Override
    public Mono<Department> findByName(String name) {
        return this.client.sql("%s WHERE d.name = :name".formatted(SELECT_QUERY))
                .bind("name", name)
                .fetch()
                .all()
                .bufferUntilChanged(result -> result.get("d_id"))
                .flatMap(Department::fromRows)
                .singleOrEmpty();
    }

    @Override
    public Mono<Department> save(Department department) {
        return this.saveDepartment(department)
                .flatMap(this::saveManager)
                .flatMap(this::saveEmployees)
                .flatMap(this::deleteDepartmentManager)
                .flatMap(this::saveDepartmentManager)
                .flatMap(this::deleteDepartmentEmployees)
                .flatMap(this::saveDepartmentEmployees);
    }

    @Override
    public Mono<Void> delete(Department department) {
        return this.deleteDepartmentManager(department)
                .flatMap(this::deleteDepartmentEmployees)
                .flatMap(this::deleteDepartment)
                .then();
    }

    private Mono<Department> saveDepartment(Department department) {
        if (department.getId() == null) {
            return this.client.sql("""
                            INSERT INTO departments(name)
                            VALUES(:name)
                            """)
                    .bind("name", department.getName())
                    .filter((statement, next) -> statement.returnGeneratedValues("id").execute())
                    .fetch()
                    .first()
                    .doOnNext(result -> department.setId(Long.parseLong(result.get("id").toString())))
                    .thenReturn(department);
        }
        return this.client.sql("""
                        UPDATE departments
                        SET name = :name
                        WHERE id = :departmentId
                        """)
                .bind("name", department.getName())
                .bind("departmentId", department.getId())
                .fetch()
                .first()
                .thenReturn(department);
    }

    private Mono<Department> saveManager(Department department) {
        return Mono.justOrEmpty(department.getManager())
                .flatMap(this.employeeRepository::save)
                .doOnNext(department::setManager)
                .thenReturn(department);
    }

    private Mono<Department> saveEmployees(Department department) {
        return Flux.fromIterable(department.getEmployees())
                .flatMap(this.employeeRepository::save)
                .collectList()
                .doOnNext(department::setEmployees)
                .thenReturn(department);
    }

    private Mono<Department> deleteDepartmentManager(Department department) {
        final String QUERY = """
                DELETE FROM department_managers WHERE department_id = :departmentId OR employee_id = :managerId
                """;
        return Mono.just(department)
                .flatMap(dep -> client.sql(QUERY)
                        .bind("departmentId", dep.getId())
                        .bind("managerId", dep.getManager().orElseGet(() -> Employee.builder().id(0L).build()).getId())
                        .fetch()
                        .rowsUpdated())
                .thenReturn(department);
    }

    private Mono<Department> saveDepartmentManager(Department department) {
        final String QUERY = """
                INSERT INTO department_managers(department_id, employee_id)
                VALUES(:departmentId, :employeeId)
                """;

        return Mono.justOrEmpty(department.getManager())
                .flatMap(manager -> client.sql(QUERY)
                        .bind("departmentId", department.getId())
                        .bind("employeeId", manager.getId())
                        .fetch()
                        .rowsUpdated())
                .thenReturn(department);
    }

    private Mono<Department> deleteDepartmentEmployees(Department department) {
        final String QUERY = """
                DELETE FROM department_employees WHERE department_id = :departmentId OR employee_id IN (:employeeIds)
                """;

        List<Long> employeeIds = department.getEmployees().stream().map(Employee::getId).toList();

        return Mono.just(department)
                .flatMap(dep -> client.sql(QUERY)
                        .bind("departmentId", department.getId())
                        .bind("employeeIds", employeeIds.isEmpty() ? List.of(0) : employeeIds)
                        .fetch()
                        .rowsUpdated())
                .thenReturn(department);
    }

    private Mono<Department> saveDepartmentEmployees(Department department) {
        final String QUERY = """
                INSERT INTO department_employees(department_id, employee_id)
                VALUES(:departmentId, :employeeId)
                """;

        return Flux.fromIterable(department.getEmployees())
                .flatMap(employee -> client.sql(QUERY)
                        .bind("departmentId", department.getId())
                        .bind("employeeId", employee.getId())
                        .fetch()
                        .rowsUpdated())
                .collectList()
                .thenReturn(department);
    }

    private Mono<Void> deleteDepartment(Department department) {
        return this.client.sql("DELETE FROM departments WHERE id = :departmentId")
                .bind("departmentId", department.getId())
                .fetch()
                .first()
                .then();
    }
}
````

Analicemos en detalle lo que hace el método `findAll()`:

````java

@Override
public Flux<Department> findAll() {
    return this.client.sql(SELECT_QUERY)
            .fetch()
            .all()
            .bufferUntilChanged(result -> result.get("d_id"))
            .flatMap(Department::fromRows);
}
````

Primero tenemos `client.sql(SELECT_QUERY).fetch().all()`, que recupera todos los datos que solicitamos en nuestra
consulta. Dado que estamos uniendo tablas, tendremos varias filas para cada `Department`.
`.bufferUntilChanged(result -> result.get("d_id"))` recopila todas las mismas filas juntas en una
`List<Map<String, Object>`, antes de pasarla finalmente a nuestra última línea que extrae los datos y devuelve
nuestros objetos `Department`.

Utilizar `.bufferUntilChanged(result -> result.get("d_id"))` es una forma eficaz de agrupar filas por departamento
`(d_id)`. De esta manera, obtienes todos los registros relacionados con el mismo departamento en una sola emisión.

Para resumir:

- `.client.sql(SELECT_QUERY).fetch().all()`, obtenga todos los datos que solicitamos.
- `.bufferUntilChanged(result -> result.get("d_id"))`, agrupe las filas en una lista según `department.id`.
- `.flatMap(Department::fromRows)`, convierta cada conjunto de resultados en un `Department`.

## Persistiendo entidades

Recuperar una entidad es sencillo: se solicitan algunos datos y se crea un objeto a partir de ellos. Pero, ¿qué sucede
si queremos conservar los datos? Debemos conservar el departamento, el gerente, el empleado y una lista de empleados.
No compartiré todo el código para esto, ya que es bastante largo, pero explicaré la idea de cómo funciona.

La forma más sencilla de comprender la conservación de estas entidades es mediante una `cadena de pasos`.
Para nuestras entidades de ejemplo, se vería así:

- Guardar o actualizar el `Department`
- Guardar o actualizar el gerente (manager) `Employee`
- Guardar o actualizar cada empleado
- Actualizar la relación entre el `Department` y `Manager`
- Actualizar la relación entre el `Department` y `Employee`

El método público se ve exactamente como el siguiente código

````java

@Override
public Mono<Department> save(Department department) {
    return this.saveDepartment(department)
            .flatMap(this::saveManager)
            .flatMap(this::saveEmployees)
            .flatMap(this::deleteDepartmentManager)
            .flatMap(this::saveDepartmentManager)
            .flatMap(this::deleteDepartmentEmployees)
            .flatMap(this::saveDepartmentEmployees);
}
````

Pasamos el `Department` inicial por cada paso, modificando el estado a medida que avanzamos.

A continuación se muestra el primer paso de este proceso: `saveDepartment`.

````java
private Mono<Department> saveDepartment(Department department) {
    if (department.getId() == null) {
        return this.client.sql("""
                        INSERT INTO departments(name)
                        VALUES(:name)
                        """)
                .bind("name", department.getName())
                .filter((statement, next) -> statement.returnGeneratedValues("id").execute())
                .fetch()
                .first()
                .doOnNext(result -> department.setId(Long.parseLong(result.get("id").toString())))
                .thenReturn(department);
    }
    return this.client.sql("""
                    UPDATE departments
                    SET name = :name
                    WHERE id = :departmentId
                    """)
            .bind("name", department.getName())
            .bind("departmentId", department.getId())
            .fetch()
            .first()
            .thenReturn(department);
}
````

Vemos que hay dos ramas, una para persistir una nueva entidad y otra para actualizarla. En la primera, usamos nuestro
cliente para insertar un nuevo `Department`, devolviendo un `id`. Luego, establecemos el `id` de nuestro objeto
`Department` antes de devolverlo para el siguiente paso. Cada paso posterior hará lo mismo, persistirá una entidad y la
establecerá nuevamente como nuestra entidad principal.

Una vez que se haya persistido el `Department`, podemos pasar a las otras entidades anidadas. Cada entidad anidada
requiere tres pasos, que verá a continuación para el administrador del departamento.

Primero persistimos el administrador. El uso de `EmployeeRepository` facilita esto, ya que no tenemos que decidir entre
una inserción o una actualización como lo hacemos con el `Department`. Una vez que se persiste, establecemos el objeto
de administrador del Departamento en el Empleado nuevo/actualizado.

````java
private Mono<Department> saveManager(Department department) {
    return Mono.justOrEmpty(department.getManager())
            .flatMap(this.employeeRepository::save)
            .doOnNext(department::setManager)
            .thenReturn(department);
}
````

Una vez que se ha conservado la entidad, queremos conservar la relación entre el departamento y el empleado. Primero
debemos eliminar cualquier relación existente.

````java
private Mono<Department> deleteDepartmentManager(Department department) {
    final String QUERY = """
            DELETE FROM department_managers WHERE department_id = :departmentId OR employee_id = :managerId
            """;
    return Mono.just(department)
            .flatMap(dep -> client.sql(QUERY)
                    .bind("departmentId", dep.getId())
                    .bind("managerId", dep.getManager().orElseGet(() -> Employee.builder().id(0L).build()).getId())
                    .fetch()
                    .rowsUpdated())
            .thenReturn(department);
}
````

Luego, debemos mantener la nueva relación. Una característica útil de `Mono` es que devuelve de `0 a 1` objetos, por lo
que si el `manager` está vacío, no se llama a `.flatMap` y pasamos a devolver el `Department` al final del método.

````java
private Mono<Department> saveDepartmentManager(Department department) {
    final String QUERY = """
            INSERT INTO department_managers(department_id, employee_id)
            VALUES(:departmentId, :employeeId)
            """;

    return Mono.justOrEmpty(department.getManager())
            .flatMap(manager -> client.sql(QUERY)
                    .bind("departmentId", department.getId())
                    .bind("employeeId", manager.getId())
                    .fetch()
                    .rowsUpdated())
            .thenReturn(department);
}
````

Para resumir, persistimos nuestro `Department`, persistimos cualquier entidad anidada `(Empleado)` y luego creamos una
relación entre los dos.

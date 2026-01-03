## Работа с join'ами, статистикой
### Создание таблиц и данных

Создадим нужные нам для работы таблицы и наполним их данными
```
-- Таблица сотрудников
CREATE TABLE employees (
    employee_id SERIAL PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50),
    department_id INT
);
-- Таблица отделов
CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    department_name VARCHAR(100)
);
-- Таблица проектов
CREATE TABLE projects (
    project_id SERIAL PRIMARY KEY,
    project_name VARCHAR(100),
    employee_id INT
);
```
Теперь данные
```
INSERT INTO employees (first_name, last_name, department_id) VALUES ('Иван', 'Иванов', 1),('Пётр', 'Петров', 2),('Анна', 'Сидорова', 1),('Мария', 'Козлова', NULL);

INSERT INTO departments (department_name) VALUES ('IT'),('HR'),('Финансы');

INSERT INTO projects (project_name, employee_id) VALUES ('Проект 1', 1),('Проект 2', 2),('Проект 3', NULL);
```

### Запросы

#### Внутреннее соединение (INNER JOIN)
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id;
```
Запрос вернёт сотрудников, у которых указан отдел.

#### Левостороннее соединение (LEFT JOIN)
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.department_id;
```
Все сотрудники будут в результате, даже если у них нет указанного отдела. Для таких сотрудников department_name будет NULL.

#### Правостороннее соединение (RIGHT JOIN)
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.department_id;
```
Все отделы будут в результате, даже если в них нет сотрудников. Для отделов без сотрудников first_name и last_name будут NULL.
#### Кросс-соединение (CROSS JOIN)
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
CROSS JOIN departments d;
```
Этот запрос аналогичен следующему
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e, departments d;
```
Каждая запись из employees соединится с каждой записью из departments. 
Если в employees 4 записи, а в departments 3, результат будет содержать 12 строк.
#### Полное соединение (FULL JOIN)
```
SELECT e.first_name, e.last_name, d.department_name
FROM employees e
FULL JOIN departments d ON e.department_id = d.department_id;
```
В результате будут:
	все сотрудники (даже без отдела);
	все отделы (даже без сотрудников).
#### Комбинированный запрос с разными типами соединений
```
SELECT e.first_name, e.last_name, d.department_name, p.project_name
FROM employees e
JOIN departments d ON e.department_id = d.department_id
LEFT JOIN projects p ON e.employee_id = p.employee_id;
```
Выбираем сотрудников, у которых есть отделы.
Если проект отсутсвует будет NULL в соответствующем поле.
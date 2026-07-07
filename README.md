# Class 10 — Spring MVC with MySQL + DAO + CRUD

**Date:** 18th June 2026 (Thursday)
**Pages:** 49–72 from PDF

---

## Section 1 — Quick Recap

> **Say to students:** "Last class we built a Spring MVC app — form, controller, JSP, Model. Today we connect it to MySQL database. We will save student data to DB, fetch all students and display in a table. This is the most important class — real project level work."

**Ask 2 questions:**
- Q1. What is Model in Spring MVC? How do you pass data to JSP?
- Q2. What does View Resolver do?

---

## Section 2 — What We Are Building

> **Say:** "We build a Student Registration system. User fills form → data saves to MySQL → all students show in a table with Delete button."

### Application Flow:

```
index.jsp (form)
    ↓ POST /save
StudentController.save()
    ↓ calls
StudentDAO.registration()
    ↓ SQL INSERT into MySQL
redirect to /studentsdata
    ↓ GET /studentsdata
StudentController.studentsdata()
    ↓ calls
StudentDAO.getallstudents()
    ↓ SQL SELECT from MySQL
studentsdata.jsp (shows table with all students)
```

### Full Project Structure:

```
SpringMVCApp/
├── src/main/java/
│   └── com/demo/
│       ├── controllers/
│       │   └── StudentController.java
│       ├── dao/
│       │   └── StudentDAO.java
│       └── models/
│           └── Student.java
├── src/main/webapp/
│   └── WEB-INF/
│       ├── pages/
│       │   ├── index.jsp
│       │   ├── studentsdata.jsp
│       │   └── welcome.jsp
│       ├── web.xml
│       └── spring-servlet.xml
└── pom.xml
```

---

## Section 3 — MySQL Setup

> **Say:** "First create the database and table in MySQL. Open MySQL Workbench or command line."

```sql
-- Create database
CREATE DATABASE springmaven1;

-- Use database
USE springmaven1;

-- Create student table
CREATE TABLE student (
    id INT AUTO_INCREMENT PRIMARY KEY,
    full_name VARCHAR(100),
    email VARCHAR(100),
    password VARCHAR(100),
    address VARCHAR(255)
);
```

> **Check:** Run `SHOW TABLES;` — should see student table.

---

## Section 4 — Update pom.xml

> **Say:** "Make sure pom.xml has MySQL connector and spring-jdbc. If added in Class 8 skip this step."

```xml
<!-- MySQL Connector -->
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>8.0.33</version>
</dependency>

<!-- Spring JDBC — for JdbcTemplate -->
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-jdbc</artifactId>
    <version>5.3.32</version>
</dependency>

<!-- JSTL — for c:forEach in JSP -->
<dependency>
    <groupId>javax.servlet</groupId>
    <artifactId>jstl</artifactId>
    <version>1.2</version>
</dependency>
```

---

## Section 5 — Update spring-servlet.xml (Add DataSource + JdbcTemplate + DAO)

> **Say:** "We add 3 new beans in spring-servlet.xml — DataSource (MySQL connection), JdbcTemplate (runs SQL), StudentDAO (our DAO class that uses JdbcTemplate)."

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans
    xmlns="http://www.springframework.org/schema/beans"
    xmlns:mvc="http://www.springframework.org/schema/mvc"
    xmlns:context="http://www.springframework.org/schema/context"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="
        http://www.springframework.org/schema/beans
        http://www.springframework.org/schema/beans/spring-beans.xsd
        http://www.springframework.org/schema/context
        http://www.springframework.org/schema/context/spring-context.xsd
        http://www.springframework.org/schema/mvc
        http://www.springframework.org/schema/mvc/spring-mvc.xsd">

    <context:component-scan base-package="com.demo.controllers" />

    <!-- View Resolver -->
    <bean class="org.springframework.web.servlet.view.InternalResourceViewResolver">
        <property name="prefix" value="/WEB-INF/pages/" />
        <property name="suffix" value=".jsp" />
    </bean>

    <!-- DataSource — MySQL connection details -->
    <bean id="dataSource"
        class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.cj.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://localhost:3306/springmaven1" />
        <property name="username" value="root" />
        <property name="password" value="your_mysql_password" />
    </bean>

    <!-- JdbcTemplate — uses DataSource to run SQL queries -->
    <bean id="jdbcTemplate"
        class="org.springframework.jdbc.core.JdbcTemplate">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- StudentDAO — our data access class, uses JdbcTemplate -->
    <bean id="dao" class="com.demo.dao.StudentDAO">
        <property name="template" ref="jdbcTemplate" />
    </bean>

    <mvc:annotation-driven />

</beans>
```

### Every new bean explained:

| Bean | Purpose |
|------|---------|
| `dataSource` | Holds MySQL connection info — driver, url, username, password |
| `driverClassName` | MySQL JDBC driver class — `com.mysql.cj.jdbc.Driver` |
| `url` | MySQL connection URL — port 3306, database springmaven1 |
| `jdbcTemplate` | Spring class that runs SQL queries — uses dataSource |
| `dao` | Our StudentDAO class — injected with jdbcTemplate |

> **Important:** Change `your_mysql_password` to your actual MySQL password.

---

## Section 6 — File 1: Student.java (Model)

> **Say:** "Student is our Model class — maps to student table in DB. One field per column."

### Create package: `com.demo.models`
### Create class: `Student`

```java
package com.demo.models;

public class Student {

    private int id;
    private String fullname;
    private String email;
    private String password;
    private String address;

    // Getters
    public int getId() { return id; }
    public String getFullname() { return fullname; }
    public String getEmail() { return email; }
    public String getPassword() { return password; }
    public String getAddress() { return address; }

    // Setters
    public void setId(int id) { this.id = id; }
    public void setFullname(String fullname) { this.fullname = fullname; }
    public void setEmail(String email) { this.email = email; }
    public void setPassword(String password) { this.password = password; }
    public void setAddress(String address) { this.address = address; }
}
```

---

## Section 7 — File 2: StudentDAO.java

> **Say:** "DAO = Data Access Object. This class handles ALL database operations. It uses JdbcTemplate to run SQL. Controller calls DAO — DAO talks to DB. Never write SQL in Controller."

### Create package: `com.demo.dao`
### Create class: `StudentDAO`

```java
package com.demo.dao;

import java.util.List;
import org.springframework.jdbc.core.JdbcTemplate;
import com.demo.models.Student;

public class StudentDAO {

    // JdbcTemplate — injected by Spring from spring-servlet.xml
    JdbcTemplate template;

    // Setter — Spring calls this to inject JdbcTemplate
    public void setTemplate(JdbcTemplate template) {
        this.template = template;
    }

    // INSERT — save new student to database
    public boolean registration(Student s) {
        String sql = "INSERT INTO student(full_name, email, password, address) VALUES(?,?,?,?)";
        template.update(sql,
            s.getFullname(),
            s.getEmail(),
            s.getPassword(),
            s.getAddress());
        return true;
    }

    // SELECT ALL — fetch all students from database
    public List<Student> getallstudents() {
        String sql = "SELECT * FROM student ORDER BY id DESC";

        return template.query(sql, (rs, rowNum) -> {
            // rs = ResultSet — one row from DB
            // rowNum = row number
            Student s = new Student();
            s.setId(rs.getInt("id"));
            s.setFullname(rs.getString("full_name"));
            s.setEmail(rs.getString("email"));
            s.setPassword(rs.getString("password"));
            s.setAddress(rs.getString("address"));
            return s;
        });
    }

    // DELETE — remove student by id
    public void delete(int id) {
        String sql = "DELETE FROM student WHERE id=?";
        template.update(sql, id);
    }
}
```

### Line by line explanation:

| Code | Explanation |
|------|-------------|
| `JdbcTemplate template` | Field — Spring injects this from spring-servlet.xml |
| `setTemplate()` | Setter — Spring uses setter injection to inject JdbcTemplate |
| `template.update(sql, args)` | Runs INSERT, UPDATE, DELETE queries |
| `?` in SQL | Placeholder — prevents SQL injection |
| `template.query(sql, rowMapper)` | Runs SELECT query — maps each row to Student object |
| `rs.getInt("id")` | Gets int value of "id" column from result row |
| `rs.getString("full_name")` | Gets String value of "full_name" column |
| Lambda `(rs, rowNum) -> {}` | Row mapper — converts one DB row to one Student object |

---

## Section 8 — File 3: StudentController.java

> **Say:** "Controller now has DAO injected using @Autowired. It calls DAO methods and passes results to JSP."

```java
package com.demo.controllers;

import java.util.List;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.ModelAttribute;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;

import com.demo.dao.StudentDAO;
import com.demo.models.Student;

@Controller
public class StudentController {

    // @Autowired — Spring automatically injects StudentDAO bean
    // defined in spring-servlet.xml as id="dao"
    @Autowired
    StudentDAO sdao;

    // GET "/" — open registration form
    @GetMapping("/")
    public String home() {
        return "index";
    }

    // POST "/save" — save student to database
    @PostMapping("/save")
    public String save(@ModelAttribute Student s) {
        // @ModelAttribute binds form fields to Student object automatically
        // fullname → s.setFullname(), email → s.setEmail() etc.

        boolean result = sdao.registration(s);

        if(result) {
            System.out.println("Registration Successfully Completed");
            return "redirect:/studentsdata";
        } else {
            System.out.println("Registration Failed");
            return "redirect:/";
        }
    }

    // GET "/studentsdata" — fetch all students and show in JSP
    @GetMapping("/studentsdata")
    public String studentsdata(Model m) {
        List<Student> list = sdao.getallstudents();
        // Add list to Model — JSP will read it as ${list}
        m.addAttribute("list", list);
        return "studentsdata";
    }

    // GET "/deletestudent/{id}" — delete student by id
    @GetMapping("/deletestudent/{id}")
    public String deletestudent(@PathVariable int id) {
        // @PathVariable gets id from URL
        // Example: /deletestudent/5 → id=5
        sdao.delete(id);
        return "redirect:/studentsdata";
    }
}
```

### New annotations explained:

| Annotation | Explanation |
|-----------|-------------|
| `@Autowired` | Spring automatically injects the matching bean — no need to write setter |
| `@ModelAttribute Student s` | Binds form fields to Student object — fullname → setFullname() automatically |
| `@PathVariable int id` | Gets value from URL path — /deletestudent/5 → id=5 |
| `redirect:/studentsdata` | After save or delete — refresh the students list page |

---

## Section 9 — File 4: index.jsp

```html
<%@ page language="java"
    contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<!DOCTYPE html>
<html>
<head><title>Student Registration</title></head>
<body>

    <h2>Student Registration Form</h2>

    <!--
        action="save" — maps to @PostMapping("/save")
        method="post" — HTTP POST
        Field names must match Student.java field names
        for @ModelAttribute to work
    -->
    <form action="save" method="post">

        Full Name:
        <input type="text" name="fullname"
               placeholder="Enter full name" /><br/><br/>

        Email:
        <input type="email" name="email"
               placeholder="Enter email" /><br/><br/>

        Password:
        <input type="password" name="password"
               placeholder="Enter password" /><br/><br/>

        Address:
        <textarea name="address"
               placeholder="Enter address"></textarea><br/><br/>

        <button type="submit">Submit</button>

    </form>

</body>
</html>
```

> **Important:** Form field names (fullname, email, password, address) must exactly match Student.java field names so @ModelAttribute can bind them automatically.

---

## Section 10 — File 5: studentsdata.jsp

> **Say:** "This page shows all students in a table. We use JSTL c:forEach to loop through the list. Each row has a Delete link."

```html
<%@ page language="java"
    contentType="text/html; charset=UTF-8"
    pageEncoding="UTF-8"%>
<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
<!DOCTYPE html>
<html>
<head><title>Students Data</title></head>
<body>

    <h1>Students Data</h1>

    <a href="/">Add New Student</a><br/><br/>

    <table border="1">
        <tr>
            <th>Id</th>
            <th>Full Name</th>
            <th>Email</th>
            <th>Password</th>
            <th>Address</th>
            <th>Action</th>
        </tr>

        <!--
            c:forEach — loops through "list" from Model
            var="s" — each Student object
            varStatus="status" — gives index, count etc.
        -->
        <c:forEach var="s" items="${list}" varStatus="status">
        <tr>
            <td>${status.index + 1}</td>
            <td>${s.fullname}</td>
            <td>${s.email}</td>
            <td>${s.password}</td>
            <td>${s.address}</td>
            <td>
                <!--
                    deletestudent/${s.id}
                    maps to @GetMapping("/deletestudent/{id}")
                    s.id = actual database id of this student
                -->
                <a href="deletestudent/${s.id}">Delete</a>
            </td>
        </tr>
        </c:forEach>

    </table>

</body>
</html>
```

### JSTL tags explained:

| Tag / Expression | Explanation |
|-----------------|-------------|
| `<%@ taglib uri="..." prefix="c" %>` | Import JSTL core tag library — needed for c: tags |
| `<c:forEach var="s" items="${list}">` | Loop through list from Model — each iteration s = one Student |
| `${s.fullname}` | Read fullname field of current Student — calls s.getFullname() |
| `${status.index + 1}` | Row number starting from 1 |
| `${s.id}` | Student id — used in delete URL |
| `deletestudent/${s.id}` | Delete URL — Example: deletestudent/5 |

---

## Section 11 — Run and Expected Output

### Run:
1. Right click project → Run As → Run on Server
2. Open browser: `http://localhost:8080/SpringMVCApp/`

### Step 1 — Registration form opens:
```
Student Registration Form
Full Name: [_________]
Email:     [_________]
Password:  [_________]
Address:   [_________]
           [Submit]
```

### Step 2 — Fill and submit → redirects to /studentsdata:
```
Students Data
Add New Student

| Id | Full Name    | Email             | Password | Address  |        |
|----|-------------|-------------------|----------|----------|--------|
| 1  | Rahul Kumar | rahul@gmail.com   | 1234     | Pune     | Delete |
| 2  | Priya Shah  | priya@gmail.com   | 5678     | Mumbai   | Delete |
```

### Step 3 — Click Delete → student removed → page refreshes

### Console output:
```
Registration Successfully Completed
```

---

## Section 12 — What Each Layer Does

> **Draw on board:**

```
Browser (JSP)
    ↕ HTTP Request/Response
Controller (@Controller)
    ↕ calls methods
DAO (StudentDAO)
    ↕ runs SQL
Database (MySQL)
```

| Layer | File | Job |
|-------|------|-----|
| View | index.jsp, studentsdata.jsp | Shows UI to user |
| Controller | StudentController.java | Handles requests, calls DAO |
| DAO | StudentDAO.java | Runs SQL queries using JdbcTemplate |
| Model | Student.java | Carries data between layers |
| Config | spring-servlet.xml | Wires everything together |

---

## Section 13 — Common Errors and Fixes

| Error | Reason | Fix |
|-------|--------|-----|
| Communications link failure | MySQL not running | Start MySQL service |
| Access denied for user root | Wrong password in spring-servlet.xml | Update password property |
| Table student not found | Table not created in DB | Run CREATE TABLE SQL |
| ${s.fullname} shows blank | Field name mismatch | Form name="fullname" must match Student field fullname |
| c:forEach not working | JSTL taglib missing | Add <%@ taglib %> at top of JSP |
| NullPointerException on sdao | @Autowired not finding DAO bean | Check bean id="dao" in spring-servlet.xml and DAO class path |

---

## Section 14 — Revision Questions

- Q1. What is DAO pattern? Why do we use it?
- Q2. What is JdbcTemplate? What methods does it have?
- Q3. What is DataSource? What 4 properties does it need?
- Q4. What does @ModelAttribute do? What must form field names match?
- Q5. What does @PathVariable do? Give example URL.
- Q6. What is the difference between template.update() and template.query()?
- Q7. How does c:forEach work in JSP?
- Q8. What is the ? in SQL queries used for?
- Q9. Write the full flow — from form submit to data showing in table.
- Q10. What layers does our application have? Name each file.

---

> **Next Class — June 20 (Class 11):** Full CRUD — Update student (edit form + update query). Complete the CRUD operations. Come with today's app running with insert and delete working.

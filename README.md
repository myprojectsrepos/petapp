# 🐾 Pet Management Application

## Overview
An enterprise-ready Java Swing + EJB pet tracker with real-time vaccination alerts, full CRUD interface, and JMS message logging.

## Modules
- **petapp-client**: Swing UI
- **petapp-ejb**: EJB logic, Hibernate, JMS, Timer
- **petapp-ear**: EAR wrapper for WildFly deployment

## Deployment
1. Deploy EAR to WildFly
2. Access `PetAppClient.jnlp` via browser at:

http://localhost:8080/PetAppClient.jnlp

3. Run JNLP, login as `admin / password123`

## Features
- ✅ Login and authentication
- ✅ Edit pet/owner/vaccine details
- ✅ UI validation (length, phone, dates)
- ✅ Timer-based vaccine expiry alerts
- ✅ JMS notifications logged to `server.log`
- ✅ H2 + Flyway-powered schema

## Flyway Scripts
- `V1__schema.sql`: Creates pet, owner, vaccine, login tables
- `V2__data.sql`: Inserts sample data

Pet.java – petapp-ejb/src/main/java/com/petapp/model/Pet.java
package com.petapp.model;

import javax.persistence.*;
import java.time.LocalDateTime;
import java.util.List;

@Entity
public class Pet {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;
    private int age;

    @Column(name = "last_vaccination")
    private LocalDateTime lastVaccination;

    @ManyToOne
    private Owner owner;

    @OneToMany(mappedBy = "pet", cascade = CascadeType.ALL)
    private List<Vaccine> vaccines;

    // Getters & Setters
}


Owner.java – petapp-ejb/src/main/java/com/petapp/model/Owner.java
package com.petapp.model;

import javax.persistence.*;
import java.util.List;

@Entity
public class Owner {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    private String phone;

    @OneToMany(mappedBy = "owner", cascade = CascadeType.ALL)
    private List<Pet> pets;

    // Getters & Setters
}

Vaccine.java – petapp-ejb/src/main/java/com/petapp/model/Vaccine.java
package com.petapp.model;

import javax.persistence.*;

@Entity
public class Vaccine {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String type;

    @ManyToOne
    private Pet pet;

    // Getters & Setters
}

UserLogin.java – petapp-ejb/src/main/java/com/petapp/model/UserLogin.java
package com.petapp.model;

import javax.persistence.Entity;
import javax.persistence.Id;

@Entity
public class UserLogin {
    @Id
    private String username;

    private String password;

    // Getters & Setters
}


V1__schema.sql – petapp-ejb/src/main/resources/db/migration/V1__schema.sql
CREATE TABLE owner (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255),
  phone VARCHAR(50)
);

CREATE TABLE pet (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(255),
  age INT,
  last_vaccination TIMESTAMP,
  owner_id BIGINT,
  FOREIGN KEY (owner_id) REFERENCES owner(id)
);

CREATE TABLE vaccine (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  type VARCHAR(255),
  pet_id BIGINT,
  FOREIGN KEY (pet_id) REFERENCES pet(id)
);

CREATE TABLE user_login (
  username VARCHAR(100) PRIMARY KEY,
  password VARCHAR(100)
);


V2__data.sql – petapp-ejb/src/main/resources/db/migration/V2__data.sql
INSERT INTO owner (name, phone) VALUES ('Sita Rao', '555-1234');

INSERT INTO pet (name, age, last_vaccination, owner_id)
VALUES ('Tommy', 4, '2025-07-21 10:00:00', 1);

INSERT INTO vaccine (type, pet_id) VALUES ('Rabies', 1);
INSERT INTO vaccine (type, pet_id) VALUES ('Distemper', 1);

INSERT INTO user_login (username, password) VALUES ('admin', 'password123');


PetServiceBean.java – petapp-ejb/src/main/java/com/petapp/ejb/PetServiceBean.java
package com.petapp.ejb;

import com.petapp.model.Pet;
import com.petapp.model.UserLogin;
import javax.ejb.Stateless;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import java.util.List;

@Stateless
public class PetServiceBean {

    @PersistenceContext
    private EntityManager em;

    public boolean authenticate(String username, String password) {
        UserLogin user = em.find(UserLogin.class, username);
        return user != null && user.getPassword().equals(password);
    }

    public List<Pet> getAllPets() {
        return em.createQuery("SELECT p FROM Pet p", Pet.class).getResultList();
    }

    public List<Pet> searchPets(String keyword) {
        return em.createQuery("SELECT p FROM Pet p WHERE LOWER(p.name) LIKE :kw", Pet.class)
                 .setParameter("kw", "%" + keyword.toLowerCase() + "%")
                 .getResultList();
    }

    public void updatePet(Pet pet) {
        em.merge(pet);
    }

    public List<Pet> getPetsNeedingVaccinationCheck() {
        return em.createQuery("SELECT p FROM Pet p WHERE p.lastVaccination < CURRENT_TIMESTAMP - 1/24.0", Pet.class)
                 .getResultList();
    }
}

VaccinationScheduler.java – petapp-ejb/src/main/java/com/petapp/ejb/VaccinationScheduler.java
package com.petapp.ejb;

import com.petapp.model.Pet;
import javax.ejb.*;
import javax.inject.Inject;
import javax.jms.*;
import java.util.List;

@Singleton
@Startup
public class VaccinationScheduler {

    @Inject
    PetServiceBean petService;

    @Resource(lookup = "java:/jms/queue/NotifyQueue")
    private Queue notifyQueue;

    @Resource
    private ConnectionFactory connectionFactory;

    @Schedule(hour = "*", minute = "*", second = "0", persistent = false)
    public void checkVaccinationTimes() {
        List<Pet> pets = petService.getPetsNeedingVaccinationCheck();
        try (JMSContext context = connectionFactory.createContext()) {
            for (Pet pet : pets) {
                String msg = "Vaccination overdue for: " + pet.getName() + ", Owner Phone: " + pet.getOwner().getPhone();
                context.createProducer().send(notifyQueue, msg);
            }
        }
    }
}

VaccinationScheduler.java – petapp-ejb/src/main/java/com/petapp/ejb/VaccinationScheduler.java
package com.petapp.ejb;

import com.petapp.model.Pet;
import javax.ejb.*;
import javax.inject.Inject;
import javax.jms.*;
import java.util.List;

@Singleton
@Startup
public class VaccinationScheduler {

    @Inject
    PetServiceBean petService;

    @Resource(lookup = "java:/jms/queue/NotifyQueue")
    private Queue notifyQueue;

    @Resource
    private ConnectionFactory connectionFactory;

    @Schedule(hour = "*", minute = "*", second = "0", persistent = false)
    public void checkVaccinationTimes() {
        List<Pet> pets = petService.getPetsNeedingVaccinationCheck();
        try (JMSContext context = connectionFactory.createContext()) {
            for (Pet pet : pets) {
                String msg = "Vaccination overdue for: " + pet.getName() + ", Owner Phone: " + pet.getOwner().getPhone();
                context.createProducer().send(notifyQueue, msg);
            }
        }
    }
}

JMSConsumer.java – petapp-ejb/src/main/java/com/petapp/ejb/JMSConsumer.java
package com.petapp.ejb;

import javax.ejb.ActivationConfigProperty;
import javax.ejb.MessageDriven;
import javax.jms.Message;
import javax.jms.TextMessage;

@MessageDriven(
    activationConfig = {
        @ActivationConfigProperty(propertyName = "destinationType", propertyValue = "javax.jms.Queue"),
        @ActivationConfigProperty(propertyName = "destinationLookup", propertyValue = "java:/jms/queue/NotifyQueue")
    }
)
public class JMSConsumer implements javax.jms.MessageListener {
    public void onMessage(Message msg) {
        try {
            if (msg instanceof TextMessage) {
                String info = ((TextMessage) msg).getText();
                System.out.println("🔔 [Vaccination Alert] " + info);
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}

petapp-ejb/src/main/resources/application.properties:
spring.datasource.url=jdbc:h2:mem:petdb;DB_CLOSE_DELAY=-1
spring.datasource.username=sa
spring.datasource.password=
spring.jpa.hibernate.ddl-auto=none
spring.flyway.locations=classpath:db/migration


LoginFrame.java
package com.petapp.client;

import com.petapp.ejb.PetServiceBean;

import javax.naming.InitialContext;
import javax.swing.*;
import java.awt.*;

public class LoginFrame extends JFrame {
    JTextField usernameField = new JTextField(15);
    JPasswordField passwordField = new JPasswordField(15);

    public LoginFrame() {
        setTitle("🐾 Pet App Login");

        JButton loginBtn = new JButton("Login");
        loginBtn.addActionListener(e -> {
            try {
                InitialContext ctx = new InitialContext();
                PetServiceBean service = (PetServiceBean) ctx.lookup("java:global/petapp-ear/petapp-ejb/PetServiceBean");

                String user = usernameField.getText();
                String pass = new String(passwordField.getPassword());
                if (service.authenticate(user, pass)) {
                    new DashboardFrame(service).setVisible(true);
                    dispose();
                } else {
                    JOptionPane.showMessageDialog(this, "Invalid username or password");
                }
            } catch (Exception ex) {
                ex.printStackTrace();
            }
        });

        setLayout(new GridLayout(3, 2));
        add(new JLabel("Username")); add(usernameField);
        add(new JLabel("Password")); add(passwordField);
        add(loginBtn);

        pack(); setLocationRelativeTo(null); setDefaultCloseOperation(EXIT_ON_CLOSE);
    }
}

wildfly/standalone/welcome-content/PetAppClient.jnlp
<?xml version="1.0" encoding="UTF-8"?>
<jnlp spec="1.0+" codebase="http://localhost:8080/" href="PetAppClient.jnlp">
  <information>
    <title>Pet Management App</title>
    <vendor>Pradeep</vendor>
    <description>Java Pet App - Swing Client</description>
  </information>
  <security>
    <all-permissions/>
  </security>
  <resources>
    <j2se version="1.8+"/>
    <jar href="PetAppClient.jar" main="true"/>
  </resources>
  <application-desc main-class="com.petapp.client.LoginFrame"/>
</jnlp>


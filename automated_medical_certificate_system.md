# Automated Medical Certificate System - Complete Project Documentation

## 1. Project Requirements & Overview

### 1.1. Project Overview
The Automated Medical Certificate System is a web-based application designed for school and company clinic nurse offices to streamline, automate, and secure the issuance of Medical Certificates. It replaces manual handwriting with a structured digital workflow, drastically reducing processing time and maintaining a centralized digital archive of all issued medical records.

### 1.2. Objectives
* **Speed:** Generate a printable and downloadable PDF Medical Certificate efficiently from backend service templates in under 2 minutes.
* **Accuracy:** Pre-populate patient details (name, student/employee ID, department/course) from a secure relational database to eliminate data entry mistakes.
* **Security & Compliance:** Restrict access exclusively to authorized clinic staff (nurses/physicians) using Spring Security and Role-Based Access Control (RBAC).
* **Audit Trail:** Maintain secure database logs of who issued each medical certificate and when.

### 1.3. Scope of Work
* **In-Scope:**
  * User authentication and role management (Nurse/Admin roles).
  * Patient search and profile lookup module.
  * Medical certificate form with dynamic data inputs (Diagnosis, Vital signs, Fit/Unfit status, Recommendations).
  * Server-side PDF generation using Java.
  * Dashboard for viewing, searching, and exporting past medical certificate records.
* **Out-of-Scope:**
  * Integration with external hospital electronic health record (EHR) systems.
  * Biometric fingerprint scanner integration (manual login preferred for simplicity).

---

## 2. Technical Requirements

### 2.1. System Architecture
* **Type:** Monolithic Web Application (Spring Boot Backend with Thymeleaf or REST API architecture).
* **Architecture Pattern:** Layered Architecture (Controller -> Service -> Repository -> Database).

### 2.2. Technology Stack (Java Ecosystem)
* **Backend:** 
  * Java 17 or higher
  * Spring Boot 3.x
  * Spring Web (REST / MVC)
  * Spring Data JPA (Hibernate ORM)
  * Spring Security (Authentication, Authorization, BCrypt password hashing)
* **Database:** 
  * MySQL 8.x (Relational database for secure patient records and logs)
* **PDF Generation Engine:** 
  * Apache PDFBox OR iText (Open-source Java libraries for server-side PDF creation)

### 2.3. Non-Functional Requirements
* **Performance:** PDF rendering and database persistence completed within 3 seconds.
* **Security:** Secure password hashing (BCrypt), session management, and strict data privacy compliance for medical records.

---

## 3. Database Entities & Source Code Implementation

### 3.1. Patient Entity (`Patient.java`)
```java
package com.example.medcert.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDate;
import java.util.List;

@Entity
@Table(name = "patients")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Patient {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "student_employee_id", unique = true, nullable = false)
    private String studentEmployeeId;

    @Column(name = "first_name", nullable = false)
    private String firstName;

    @Column(name = "last_name", nullable = false)
    private String lastName;

    @Column(name = "birth_date")
    private LocalDate birthDate;

    @Column(name = "gender")
    private String gender;

    @Column(name = "department_course")
    private String departmentCourse;

    @Column(name = "contact_number")
    private String contactNumber;

    @OneToMany(mappedBy = "patient", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<MedicalCertificate> medicalCertificates;
}
```

### 3.2. Medical Certificate Entity (`MedicalCertificate.java`)
```java
package com.example.medcert.model;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.time.LocalDateTime;

@Entity
@Table(name = "medical_certificates")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class MedicalCertificate {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "certificate_number", unique = true, nullable = false)
    private String certificateNumber;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "patient_id", nullable = false)
    private Patient patient;

    @Column(name = "consultation_date", nullable = false)
    private LocalDateTime consultationDate;

    @Column(name = "diagnosis", columnDefinition = "TEXT", nullable = false)
    private String diagnosis;

    @Column(name = "recommendations", columnDefinition = "TEXT")
    private String recommendations;

    @Column(name = "fit_status")
    private String fitStatus;

    @Column(name = "issued_by")
    private String issuedBy;

    @Column(name = "created_at", updatable = false)
    private LocalDateTime createdAt = LocalDateTime.now();
}
```

### 3.3. Repositories (`PatientRepository.java` & `MedicalCertificateRepository.java`)
```java
package com.example.medcert.repository;

import com.example.medcert.model.Patient;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.Optional;
import java.util.List;

@Repository
public interface PatientRepository extends JpaRepository<Patient, Long> {
    Optional<Patient> findByStudentEmployeeId(String studentEmployeeId);
    List<Patient> findByLastNameContainingIgnoreCaseOrFirstNameContainingIgnoreCase(String lastName, String firstName);
}
```

```java
package com.example.medcert.repository;

import com.example.medcert.model.MedicalCertificate;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;
import java.util.List;
import java.util.Optional;

@Repository
public interface MedicalCertificateRepository extends JpaRepository<MedicalCertificate, Long> {
    List<MedicalCertificate> findByPatientId(Long patientId);
    Optional<MedicalCertificate> findByCertificateNumber(String certificateNumber);
}
```

### 3.4. Services (`PatientService.java` & `MedicalCertificateService.java`)
```java
package com.example.medcert.service;

import com.example.medcert.model.Patient;
import com.example.medcert.repository.PatientRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Optional;

@Service
public class PatientService {

    @Autowired
    private PatientRepository patientRepository;

    public List<Patient> getAllPatients() { return patientRepository.findAll(); }
    public Optional<Patient> getPatientById(Long id) { return patientRepository.findById(id); }
    public Optional<Patient> getPatientByStudentId(String id) { return patientRepository.findByStudentEmployeeId(id); }
    public Patient savePatient(Patient patient) { return patientRepository.save(patient); }
    public List<Patient> searchPatients(String keyword) { 
        return patientRepository.findByLastNameContainingIgnoreCaseOrFirstNameContainingIgnoreCase(keyword, keyword); 
    }
}
```

```java
package com.example.medcert.service;

import com.example.medcert.model.MedicalCertificate;
import com.example.medcert.model.Patient;
import com.example.medcert.repository.MedicalCertificateRepository;
import com.example.medcert.repository.PatientRepository;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

@Service
public class MedicalCertificateService {

    @Autowired
    private MedicalCertificateRepository medCertRepository;

    @Autowired
    private PatientRepository patientRepository;

    public List<MedicalCertificate> getAllCertificates() { return medCertRepository.findAll(); }
    public List<MedicalCertificate> getCertificatesByPatient(Long patientId) { return medCertRepository.findByPatientId(patientId); }
    public Optional<MedicalCertificate> getCertificateByNumber(String certNumber) { return medCertRepository.findByCertificateNumber(certNumber); }

    public MedicalCertificate issueCertificate(Long patientId, MedicalCertificate medCertDetails) {
        Patient patient = patientRepository.findById(patientId)
                .orElseThrow(() -> new RuntimeException("Patient not found with ID: " + patientId));

        medCertDetails.setPatient(patient);
        medCertDetails.setConsultationDate(LocalDateTime.now());
        
        String uniqueCertNo = "MED-" + LocalDateTime.now().getYear() + "-" + UUID.randomUUID().toString().substring(0, 6).toUpperCase();
        medCertDetails.setCertificateNumber(uniqueCertNo);

        return medCertRepository.save(medCertDetails);
    }
}
```

---

## 4. Development Task List (Roadmap)

- [ ] **Phase 1: Setup & Configuration**
  - [ ] Initialize Spring Boot project with Web, Data JPA, MySQL, Security, and Lombok.
  - [ ] Configure `application.properties` database connection string.
- [ ] **Phase 2: Backend Implementation**
  - [ ] Implement Entity classes and Flyway/Hibernate table auto-generation.
  - [ ] Build Repository and Service layers for Patient & MedCert.
- [ ] **Phase 3: API & PDF Generation**
  - [ ] Build REST Controllers for patient search and certificate issuance.
  - [ ] Integrate Apache PDFBox to render certificates into official document layouts.
- [ ] **Phase 4: Testing & Deployment**
  - [ ] Perform integration testing with Postman and deploy locally for nurse office review.
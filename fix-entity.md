```
In the existing doctor-service module, add two new fields to the Doctor entity: name (String, the doctor's full name) and contactNumber (String). Update DoctorRequest DTO to include both with validation: name @NotBlank, contactNumber @NotBlank @Pattern(regexp = "^[0-9+\\-\\s]{7,15}$", message = "invalid phone number format"). Update DoctorService's ModelMapper-based mapping in createDoctorProfile to carry these new fields through, and update any DoctorResponse DTO so both fields are returned in API responses.

Note: since spring.jpa.hibernate.ddl-auto=update only adds columns, it won't backfill existing rows — any Doctor records already in doctor_db will have NULL for name and contactNumber until manually updated. Flag this rather than silently ignoring it.
```

---

```
In the existing patient-service module, remove the emergencyContact field entirely from the Patient entity, from PatientRequest DTO, and from any response DTO. Add a new field contactNumber (String, @NotBlank @Pattern(regexp = "^[0-9+\\-\\s]{7,15}$", message = "invalid phone number format")) in its place. Update PatientService's createProfile and updateProfile methods and their ModelMapper mappings accordingly.

Note: ddl-auto=update does not drop columns, so the old emergency_contact column will remain in patient_db as an unused orphaned column rather than being removed — mention this explicitly rather than assuming it's gone. If a clean schema matters, that would need a manual ALTER TABLE ... DROP COLUMN or a proper migration tool like Flyway, which is out of scope for ddl-auto alone.
```

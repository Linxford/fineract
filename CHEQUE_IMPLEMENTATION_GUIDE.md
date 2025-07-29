# Apache Fineract - Cheque Management Implementation Guide

## Table of Contents
1. [Executive Summary](#executive-summary)
2. [Current Payment Architecture Analysis](#current-payment-architecture-analysis)
3. [Cheque Management System Requirements](#cheque-management-system-requirements)
4. [Implementation Architecture](#implementation-architecture)
5. [Detailed Implementation Steps](#detailed-implementation-steps)
6. [API Endpoints Design](#api-endpoints-design)
7. [Integration with Existing Modules](#integration-with-existing-modules)
8. [Security Considerations](#security-considerations)
9. [Testing Strategy](#testing-strategy)
10. [Deployment Guide](#deployment-guide)

## Executive Summary

Fineract already has basic cheque support through the `PaymentDetail` entity which includes a `checkNumber` field. However, it lacks comprehensive cheque management features like:
- Cheque book issuance and management
- Cheque status tracking (issued, presented, cleared, bounced, stopped)
- Cheque truncation system (CTS) integration
- Automated clearing and reconciliation

This guide provides a complete implementation plan to add enterprise-grade cheque management to Apache Fineract.

## Current Payment Architecture Analysis

### Existing Components

1. **PaymentDetail Entity** (`m_payment_detail` table)
   - Already has `check_number` field
   - Links to PaymentType
   - Used in loan and savings transactions

2. **PaymentType Entity** (`m_payment_type` table)
   - Defines payment methods (cash, cheque, etc.)
   - Has `is_cash_payment` flag
   - System-defined and custom types supported

3. **Transaction Processing**
   - Loan transactions use PaymentDetail
   - Savings transactions use PaymentDetail
   - Client transactions support payment details

## Cheque Management System Requirements

### Core Features (Based on Industry Standards 2024)

1. **Cheque Book Management**
   - Issue cheque books with serial numbers
   - Track cheque leaves (used/unused)
   - Multiple cheque books per account
   - Cheque book request workflow

2. **Cheque Lifecycle Management**
   - Status tracking: Unused → Issued → Presented → Cleared/Bounced
   - Stop payment requests
   - Post-dated cheque management
   - Cheque cancellation

3. **Clearing Process**
   - Support for T+1 clearing (2024 standard)
   - Cheque Truncation System (CTS) integration
   - Image-based clearing support
   - MICR code validation

4. **Reconciliation & Reporting**
   - Bank reconciliation
   - Outstanding cheque reports
   - Stale cheque management
   - Audit trail

## Implementation Architecture

### Database Schema Design

```sql
-- 1. Cheque Book Master
CREATE TABLE m_cheque_book (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    account_type VARCHAR(20) NOT NULL, -- 'SAVINGS', 'LOAN'
    account_id BIGINT NOT NULL,
    cheque_book_no VARCHAR(50) NOT NULL,
    no_of_leaves INT NOT NULL,
    start_serial_no BIGINT NOT NULL,
    end_serial_no BIGINT NOT NULL,
    issue_date DATE NOT NULL,
    status_enum SMALLINT NOT NULL, -- 1=REQUESTED, 2=ISSUED, 3=CANCELLED
    created_by BIGINT NOT NULL,
    created_date DATETIME NOT NULL,
    lastmodified_by BIGINT,
    lastmodified_date DATETIME,
    UNIQUE KEY (cheque_book_no),
    INDEX idx_account (account_type, account_id)
);

-- 2. Cheque Register (Individual Cheques)
CREATE TABLE m_cheque (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    cheque_book_id BIGINT NOT NULL,
    cheque_no BIGINT NOT NULL,
    cheque_date DATE,
    amount DECIMAL(19,6),
    payee_name VARCHAR(200),
    status_enum SMALLINT NOT NULL, -- 1=UNUSED, 2=ISSUED, 3=PRESENTED, 4=CLEARED, 5=BOUNCED, 6=STOPPED, 7=CANCELLED
    issue_date DATE,
    presented_date DATE,
    cleared_date DATE,
    stop_payment_date DATE,
    stop_payment_reason VARCHAR(500),
    transaction_id BIGINT, -- Link to actual transaction
    transaction_type VARCHAR(20), -- 'LOAN_REPAYMENT', 'WITHDRAWAL', etc.
    created_by BIGINT NOT NULL,
    created_date DATETIME NOT NULL,
    lastmodified_by BIGINT,
    lastmodified_date DATETIME,
    FOREIGN KEY (cheque_book_id) REFERENCES m_cheque_book(id),
    UNIQUE KEY (cheque_book_id, cheque_no),
    INDEX idx_status (status_enum),
    INDEX idx_dates (issue_date, presented_date, cleared_date)
);

-- 3. Cheque Status History
CREATE TABLE m_cheque_status_history (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    cheque_id BIGINT NOT NULL,
    from_status SMALLINT NOT NULL,
    to_status SMALLINT NOT NULL,
    changed_date DATETIME NOT NULL,
    changed_by BIGINT NOT NULL,
    remarks VARCHAR(500),
    FOREIGN KEY (cheque_id) REFERENCES m_cheque(id)
);

-- 4. Cheque Images (for CTS)
CREATE TABLE m_cheque_images (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    cheque_id BIGINT NOT NULL,
    image_type VARCHAR(20) NOT NULL, -- 'FRONT', 'BACK'
    image_data LONGBLOB,
    file_name VARCHAR(100),
    file_size INT,
    mime_type VARCHAR(50),
    uploaded_date DATETIME NOT NULL,
    FOREIGN KEY (cheque_id) REFERENCES m_cheque(id)
);

-- 5. Bank MICR Master
CREATE TABLE m_bank_micr (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    bank_name VARCHAR(100) NOT NULL,
    branch_name VARCHAR(100),
    micr_code VARCHAR(20) NOT NULL,
    ifsc_code VARCHAR(20),
    is_active BOOLEAN DEFAULT TRUE,
    UNIQUE KEY (micr_code)
);
```

### Domain Model Design

```java
// 1. ChequeBook Entity
@Entity
@Table(name = "m_cheque_book")
public class ChequeBook extends AbstractPersistableCustom<Long> {
    
    @Enumerated(EnumType.STRING)
    @Column(name = "account_type")
    private AccountType accountType;
    
    @Column(name = "account_id")
    private Long accountId;
    
    @Column(name = "cheque_book_no")
    private String chequeBookNumber;
    
    @Column(name = "no_of_leaves")
    private Integer numberOfLeaves;
    
    @Column(name = "start_serial_no")
    private Long startSerialNumber;
    
    @Column(name = "end_serial_no")
    private Long endSerialNumber;
    
    @Column(name = "issue_date")
    private LocalDate issueDate;
    
    @Column(name = "status_enum")
    private Integer status;
    
    @OneToMany(mappedBy = "chequeBook", fetch = FetchType.LAZY)
    private List<Cheque> cheques;
    
    // Audit fields and methods
}

// 2. Cheque Entity
@Entity
@Table(name = "m_cheque")
public class Cheque extends AbstractPersistableCustom<Long> {
    
    @ManyToOne
    @JoinColumn(name = "cheque_book_id")
    private ChequeBook chequeBook;
    
    @Column(name = "cheque_no")
    private Long chequeNumber;
    
    @Column(name = "cheque_date")
    private LocalDate chequeDate;
    
    @Column(name = "amount")
    private BigDecimal amount;
    
    @Column(name = "payee_name")
    private String payeeName;
    
    @Column(name = "status_enum")
    private Integer status;
    
    @OneToMany(mappedBy = "cheque", cascade = CascadeType.ALL)
    private List<ChequeStatusHistory> statusHistory;
    
    @OneToMany(mappedBy = "cheque", cascade = CascadeType.ALL)
    private List<ChequeImage> images;
}

// 3. Enums
public enum ChequeStatus {
    UNUSED(1, "cheque.status.unused"),
    ISSUED(2, "cheque.status.issued"),
    PRESENTED(3, "cheque.status.presented"),
    CLEARED(4, "cheque.status.cleared"),
    BOUNCED(5, "cheque.status.bounced"),
    STOPPED(6, "cheque.status.stopped"),
    CANCELLED(7, "cheque.status.cancelled");
    
    private final Integer value;
    private final String code;
}

public enum ChequeBookStatus {
    REQUESTED(1, "chequebook.status.requested"),
    ISSUED(2, "chequebook.status.issued"),
    CANCELLED(3, "chequebook.status.cancelled");
}
```

## Detailed Implementation Steps

### Step 1: Create Database Migration

Create file: `/fineract-provider/src/main/resources/db/changelog/tenant/parts/cheque_management_tables.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.3.xsd">

    <changeSet author="fineract" id="create-cheque-management-tables">
        <!-- Create cheque book table -->
        <createTable tableName="m_cheque_book">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <!-- Add all columns as defined in schema above -->
        </createTable>
        
        <!-- Create other tables -->
        <!-- Add indexes and foreign keys -->
        
        <!-- Add payment type for cheque if not exists -->
        <sql>
            INSERT INTO m_payment_type (value, description, is_cash_payment, order_position, code_name, is_system_defined)
            SELECT 'Cheque', 'Payment by cheque', 0, 2, 'paymenttype.cheque', 1
            WHERE NOT EXISTS (SELECT 1 FROM m_payment_type WHERE code_name = 'paymenttype.cheque');
        </sql>
        
        <!-- Add permissions -->
        <sql>
            INSERT INTO m_permission (grouping, code, entity_name, action_name, can_maker_checker) VALUES
            ('portfolio', 'CREATE_CHEQUEBOOK', 'CHEQUEBOOK', 'CREATE', 0),
            ('portfolio', 'UPDATE_CHEQUEBOOK', 'CHEQUEBOOK', 'UPDATE', 0),
            ('portfolio', 'DELETE_CHEQUEBOOK', 'CHEQUEBOOK', 'DELETE', 0),
            ('portfolio', 'READ_CHEQUEBOOK', 'CHEQUEBOOK', 'READ', 0),
            ('portfolio', 'ISSUE_CHEQUE', 'CHEQUE', 'ISSUE', 0),
            ('portfolio', 'STOP_CHEQUE', 'CHEQUE', 'STOP', 0),
            ('portfolio', 'CANCEL_CHEQUE', 'CHEQUE', 'CANCEL', 0),
            ('portfolio', 'CLEAR_CHEQUE', 'CHEQUE', 'CLEAR', 0);
        </sql>
    </changeSet>
</databaseChangeLog>
```

### Step 2: Create API Resources

Create file: `/fineract-provider/src/main/java/org/apache/fineract/portfolio/cheque/api/ChequeManagementApiResource.java`

```java
@Path("/v1/cheques")
@Component
@Tag(name = "Cheque Management", description = "Cheque book and cheque management")
@RequiredArgsConstructor
public class ChequeManagementApiResource {

    private final PlatformSecurityContext context;
    private final ChequeReadPlatformService readService;
    private final PortfolioCommandSourceWritePlatformService commandService;
    private final DefaultToApiJsonSerializer<ChequeData> jsonSerializer;

    // Cheque Book APIs
    @POST
    @Path("books")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Request Cheque Book", 
               description = "Request a new cheque book for an account")
    public CommandProcessingResult requestChequeBook(
            @RequestBody ChequeBookRequest request) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .requestChequeBook()
                .withJson(jsonSerializer.serialize(request))
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @GET
    @Path("books/{accountType}/{accountId}")
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "List Cheque Books", 
               description = "List all cheque books for an account")
    public List<ChequeBookData> listChequeBooks(
            @PathParam("accountType") String accountType,
            @PathParam("accountId") Long accountId) {
        context.authenticatedUser().validateHasReadPermission("CHEQUEBOOK");
        return readService.retrieveChequeBooks(accountType, accountId);
    }

    // Individual Cheque APIs
    @POST
    @Path("issue")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Issue Cheque", 
               description = "Record issuance of a cheque")
    public CommandProcessingResult issueCheque(
            @RequestBody IssueChequeRequest request) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .issueCheque()
                .withJson(jsonSerializer.serialize(request))
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @POST
    @Path("{chequeId}/stop")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Stop Cheque Payment", 
               description = "Request stop payment on a cheque")
    public CommandProcessingResult stopCheque(
            @PathParam("chequeId") Long chequeId,
            @RequestBody StopChequeRequest request) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .stopCheque(chequeId)
                .withJson(jsonSerializer.serialize(request))
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @POST
    @Path("{chequeId}/present")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Present Cheque", 
               description = "Mark cheque as presented for clearing")
    public CommandProcessingResult presentCheque(
            @PathParam("chequeId") Long chequeId,
            @RequestBody PresentChequeRequest request) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .presentCheque(chequeId)
                .withJson(jsonSerializer.serialize(request))
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @POST
    @Path("{chequeId}/clear")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Clear Cheque", 
               description = "Mark cheque as cleared")
    public CommandProcessingResult clearCheque(
            @PathParam("chequeId") Long chequeId) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .clearCheque(chequeId)
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @POST
    @Path("{chequeId}/bounce")
    @Consumes({MediaType.APPLICATION_JSON})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Bounce Cheque", 
               description = "Mark cheque as bounced")
    public CommandProcessingResult bounceCheque(
            @PathParam("chequeId") Long chequeId,
            @RequestBody BounceChequeRequest request) {
        final CommandWrapper commandRequest = new CommandWrapperBuilder()
                .bounceCheque(chequeId)
                .withJson(jsonSerializer.serialize(request))
                .build();
        return commandService.logCommandSource(commandRequest);
    }

    @GET
    @Path("outstanding")
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "List Outstanding Cheques", 
               description = "List all outstanding cheques")
    public Page<ChequeData> listOutstandingCheques(
            @QueryParam("offset") Integer offset,
            @QueryParam("limit") Integer limit) {
        context.authenticatedUser().validateHasReadPermission("CHEQUE");
        SearchParameters searchParameters = SearchParameters.builder()
                .offset(offset)
                .limit(limit)
                .build();
        return readService.retrieveOutstandingCheques(searchParameters);
    }

    @GET
    @Path("{chequeId}/status-history")
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Cheque Status History", 
               description = "Get status change history of a cheque")
    public List<ChequeStatusHistoryData> getChequeStatusHistory(
            @PathParam("chequeId") Long chequeId) {
        context.authenticatedUser().validateHasReadPermission("CHEQUE");
        return readService.retrieveChequeStatusHistory(chequeId);
    }

    // Cheque Image APIs for CTS
    @POST
    @Path("{chequeId}/images")
    @Consumes({MediaType.MULTIPART_FORM_DATA})
    @Produces({MediaType.APPLICATION_JSON})
    @Operation(summary = "Upload Cheque Image", 
               description = "Upload cheque image for CTS")
    public CommandProcessingResult uploadChequeImage(
            @PathParam("chequeId") Long chequeId,
            @FormDataParam("file") InputStream uploadedInputStream,
            @FormDataParam("file") FormDataContentDisposition fileDetail,
            @FormDataParam("type") String imageType) {
        // Implementation for image upload
    }
}
```

### Step 3: Create Service Layer

Create file: `/fineract-provider/src/main/java/org/apache/fineract/portfolio/cheque/service/ChequeWritePlatformServiceImpl.java`

```java
@Service
@RequiredArgsConstructor
@Transactional
public class ChequeWritePlatformServiceImpl implements ChequeWritePlatformService {

    private final ChequeBookRepository chequeBookRepository;
    private final ChequeRepository chequeRepository;
    private final PlatformSecurityContext context;
    private final ChequeAssembler chequeAssembler;

    @Override
    public CommandProcessingResult requestChequeBook(JsonCommand command) {
        try {
            context.authenticatedUser();
            
            // Validate request
            ChequeBookDataValidator validator = new ChequeBookDataValidator();
            validator.validateForCreate(command.json());
            
            // Create cheque book
            ChequeBook chequeBook = ChequeBook.fromJson(command);
            
            // Generate individual cheque records
            List<Cheque> cheques = new ArrayList<>();
            for (long i = chequeBook.getStartSerialNumber(); 
                 i <= chequeBook.getEndSerialNumber(); i++) {
                Cheque cheque = Cheque.createNew(chequeBook, i);
                cheques.add(cheque);
            }
            chequeBook.setCheques(cheques);
            
            // Save
            ChequeBook savedChequeBook = chequeBookRepository.saveAndFlush(chequeBook);
            
            return new CommandProcessingResultBuilder()
                    .withCommandId(command.commandId())
                    .withEntityId(savedChequeBook.getId())
                    .build();
                    
        } catch (DataIntegrityViolationException e) {
            handleDataIntegrityIssues(command, e);
            return CommandProcessingResult.empty();
        }
    }

    @Override
    public CommandProcessingResult issueCheque(JsonCommand command) {
        context.authenticatedUser();
        
        Long chequeId = command.longValueOfParameterNamed("chequeId");
        Cheque cheque = chequeRepository.findById(chequeId)
                .orElseThrow(() -> new ChequeNotFoundException(chequeId));
        
        // Validate cheque can be issued
        if (!cheque.canBeIssued()) {
            throw new InvalidChequeStateTransitionException(
                cheque.getStatus(), ChequeStatus.ISSUED);
        }
        
        // Update cheque details
        cheque.issue(command);
        
        // Create status history
        ChequeStatusHistory history = ChequeStatusHistory.create(
            cheque, cheque.getStatus(), ChequeStatus.ISSUED.getValue(), 
            "Cheque issued");
        cheque.addStatusHistory(history);
        
        chequeRepository.saveAndFlush(cheque);
        
        return new CommandProcessingResultBuilder()
                .withEntityId(cheque.getId())
                .with(cheque.getChanges())
                .build();
    }

    @Override
    public CommandProcessingResult clearCheque(Long chequeId) {
        context.authenticatedUser();
        
        Cheque cheque = chequeRepository.findById(chequeId)
                .orElseThrow(() -> new ChequeNotFoundException(chequeId));
        
        // Validate cheque can be cleared
        if (!cheque.canBeCleared()) {
            throw new InvalidChequeStateTransitionException(
                cheque.getStatus(), ChequeStatus.CLEARED);
        }
        
        // Clear the cheque
        cheque.clear();
        
        // If linked to a transaction, process it
        if (cheque.getTransactionId() != null) {
            processChequeTransaction(cheque);
        }
        
        chequeRepository.saveAndFlush(cheque);
        
        return new CommandProcessingResultBuilder()
                .withEntityId(cheque.getId())
                .build();
    }

    private void processChequeTransaction(Cheque cheque) {
        // Process the actual financial transaction
        // This would integrate with existing loan/savings transaction processing
    }
}
```

### Step 4: Create Batch Jobs for Clearing

Create file: `/fineract-provider/src/main/java/org/apache/fineract/portfolio/cheque/jobs/ChequeClearingConfig.java`

```java
@Configuration
@EnableBatchProcessing
@EnableScheduling
@ConditionalOnProperty("fineract.mode.batch-manager-enabled")
@RequiredArgsConstructor
public class ChequeClearingConfig {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final ChequeRepository chequeRepository;
    private final ChequeWritePlatformService chequeService;

    @Bean
    public Job chequeClearingJob() {
        return jobBuilderFactory.get("CHEQUE-CLEARING")
                .start(processPresentedChequesStep())
                .next(processStaleChequeStep())
                .build();
    }

    @Bean
    public Step processPresentedChequesStep() {
        return stepBuilderFactory.get("process-presented-cheques")
                .tasklet(processPresentedChequesTasklet())
                .build();
    }

    @Bean
    public Tasklet processPresentedChequesTasklet() {
        return (contribution, chunkContext) -> {
            LocalDate clearingDate = DateUtils.getBusinessLocalDate();
            
            // Get all presented cheques eligible for clearing (T+1)
            List<Cheque> presentedCheques = chequeRepository
                    .findByStatusAndPresentedDateBefore(
                        ChequeStatus.PRESENTED.getValue(), 
                        clearingDate.minusDays(1));
            
            for (Cheque cheque : presentedCheques) {
                try {
                    // Verify funds availability
                    if (verifyFundsAvailable(cheque)) {
                        chequeService.clearCheque(cheque.getId());
                    } else {
                        chequeService.bounceCheque(cheque.getId(), 
                            "Insufficient funds");
                    }
                } catch (Exception e) {
                    log.error("Error processing cheque: " + cheque.getId(), e);
                }
            }
            
            return RepeatStatus.FINISHED;
        };
    }

    @Bean
    public Step processStaleChequeStep() {
        return stepBuilderFactory.get("process-stale-cheques")
                .tasklet((contribution, chunkContext) -> {
                    // Mark cheques older than 3 months as stale
                    LocalDate staleDate = DateUtils.getBusinessLocalDate()
                            .minusMonths(3);
                    
                    List<Cheque> staleCheques = chequeRepository
                            .findByStatusAndIssueDateBefore(
                                ChequeStatus.ISSUED.getValue(), 
                                staleDate);
                    
                    for (Cheque cheque : staleCheques) {
                        cheque.markAsStale();
                        chequeRepository.save(cheque);
                    }
                    
                    return RepeatStatus.FINISHED;
                })
                .build();
    }

    private boolean verifyFundsAvailable(Cheque cheque) {
        // Implement fund verification logic
        // Check account balance based on cheque.getAccountType()
        return true;
    }
}
```

### Step 5: Integration with Existing Transactions

Modify existing transaction services to support cheque payments:

```java
// In LoanTransactionAssembler.java
public LoanTransaction assembleLoanTransaction(JsonCommand command) {
    // Existing code...
    
    // Check if payment is by cheque
    Long paymentTypeId = command.longValueOfParameterNamed("paymentTypeId");
    PaymentDetail paymentDetail = paymentDetailAssembler
            .fetchPaymentDetail(paymentTypeId, command);
    
    if (paymentDetail != null && isChequePayment(paymentTypeId)) {
        // Link to cheque record
        String chequeNumber = paymentDetail.getCheckNumber();
        Cheque cheque = chequeRepository
                .findByCheckNumber(chequeNumber)
                .orElseThrow(() -> new ChequeNotFoundException(chequeNumber));
        
        // Update cheque with transaction details
        cheque.linkToTransaction(transaction.getId(), "LOAN_REPAYMENT");
        chequeRepository.save(cheque);
    }
    
    // Continue with existing logic...
}
```

## API Endpoints Design

### Cheque Book Management
```
POST   /v1/cheques/books                    - Request new cheque book
GET    /v1/cheques/books/{accountType}/{accountId} - List cheque books
PUT    /v1/cheques/books/{bookId}          - Update cheque book
DELETE /v1/cheques/books/{bookId}          - Cancel cheque book
```

### Individual Cheque Management
```
POST   /v1/cheques/issue                    - Issue a cheque
GET    /v1/cheques/{chequeId}              - Get cheque details
POST   /v1/cheques/{chequeId}/stop         - Stop payment
POST   /v1/cheques/{chequeId}/present      - Present for clearing
POST   /v1/cheques/{chequeId}/clear        - Clear cheque
POST   /v1/cheques/{chequeId}/bounce       - Bounce cheque
POST   /v1/cheques/{chequeId}/cancel       - Cancel cheque
```

### Reporting & Queries
```
GET    /v1/cheques/outstanding              - List outstanding cheques
GET    /v1/cheques/cleared                  - List cleared cheques
GET    /v1/cheques/bounced                  - List bounced cheques
GET    /v1/cheques/{chequeId}/history      - Status history
GET    /v1/cheques/reconciliation          - Bank reconciliation report
```

### Cheque Images (CTS)
```
POST   /v1/cheques/{chequeId}/images       - Upload cheque image
GET    /v1/cheques/{chequeId}/images       - Get cheque images
DELETE /v1/cheques/{chequeId}/images/{imageId} - Delete image
```

## Integration with Existing Modules

### 1. Savings Module Integration
- Link cheque books to savings accounts
- Process cheque withdrawals
- Handle cheque deposits

### 2. Loan Module Integration
- Accept loan repayments by cheque
- Track post-dated cheques for EMI

### 3. Accounting Module Integration
- Create journal entries for cleared cheques
- Handle bounced cheque accounting
- Bank reconciliation entries

### 4. Notification Integration
- SMS/Email alerts for cheque status changes
- Alerts for bounced cheques
- Cheque book delivery notifications

## Security Considerations

1. **Access Control**
   - Role-based permissions for cheque operations
   - Maker-checker for high-value cheques
   - Audit trail for all operations

2. **Fraud Prevention**
   - Duplicate cheque number validation
   - MICR code verification
   - Signature verification integration
   - Daily limits on cheque amounts

3. **Data Security**
   - Encrypt cheque images at rest
   - Secure transmission of CTS data
   - PCI compliance for cheque details

## Testing Strategy

### Unit Tests
```java
@Test
public void testChequeIssuance() {
    // Given
    ChequeBook chequeBook = createTestChequeBook();
    Cheque cheque = chequeBook.getCheques().get(0);
    
    // When
    cheque.issue(createIssuanceCommand());
    
    // Then
    assertEquals(ChequeStatus.ISSUED.getValue(), cheque.getStatus());
    assertNotNull(cheque.getIssueDate());
    assertEquals("Test Payee", cheque.getPayeeName());
}

@Test
public void testChequeStatusTransition() {
    // Test valid transitions
    assertTrue(cheque.canTransitionTo(ChequeStatus.PRESENTED));
    
    // Test invalid transitions
    cheque.setStatus(ChequeStatus.CLEARED.getValue());
    assertFalse(cheque.canTransitionTo(ChequeStatus.UNUSED));
}
```

### Integration Tests
- Test cheque clearing batch job
- Test transaction integration
- Test reconciliation process

### E2E Tests
- Complete cheque lifecycle test
- Multi-tenant cheque processing
- Performance tests for bulk operations

## Deployment Guide

### 1. Database Migration
```bash
# Run Liquibase migration
./gradlew update -PrunList=changelog-cheque.xml
```

### 2. Configuration
```properties
# Add to application.properties
fineract.cheque.stale-period-months=3
fineract.cheque.clearing-days=1
fineract.cheque.image-max-size-mb=5
fineract.cheque.cts-enabled=true
```

### 3. Permissions Setup
```sql
-- Assign cheque permissions to roles
INSERT INTO m_role_permission (role_id, permission_id)
SELECT r.id, p.id 
FROM m_role r, m_permission p
WHERE r.name = 'Super user' 
AND p.code LIKE '%CHEQUE%';
```

### 4. Initial Data Setup
```sql
-- Add banks for MICR validation
INSERT INTO m_bank_micr (bank_name, branch_name, micr_code, ifsc_code)
VALUES 
('State Bank of India', 'Main Branch', '110002021', 'SBIN0000001'),
('HDFC Bank', 'City Center', '110240001', 'HDFC0000001');
```

### 5. Monitoring
- Set up alerts for failed cheque clearings
- Monitor cheque processing job performance
- Track cheque bounce rates

## Future Enhancements

1. **OCR Integration**
   - Automatic MICR reading
   - Handwriting recognition for amounts

2. **Blockchain Integration**
   - Immutable cheque records
   - Inter-bank clearing on blockchain

3. **AI/ML Features**
   - Fraud detection
   - Signature verification
   - Predictive bounce analysis

4. **Mobile Integration**
   - Mobile cheque deposit
   - Real-time status tracking
   - Push notifications

5. **Advanced CTS Features**
   - Real-time clearing (future RBI initiative)
   - Cross-border cheque clearing
   - Multi-currency cheque support

## Conclusion

This implementation guide provides a comprehensive framework for adding enterprise-grade cheque management to Apache Fineract. The modular design ensures easy integration with existing functionality while providing room for future enhancements. The solution supports modern banking requirements including CTS, real-time processing, and comprehensive audit trails.
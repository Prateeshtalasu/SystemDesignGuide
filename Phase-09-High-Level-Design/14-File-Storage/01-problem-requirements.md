# File Storage System (Google Drive/Dropbox) - Problem Requirements

## 1. Problem Statement

Design a cloud file storage system like Google Drive or Dropbox that allows users to store, sync, and share files across multiple devices. The system must handle file uploads, downloads, synchronization, versioning, and collaborative sharing while maintaining data integrity and security.

### What is Cloud File Storage?

Cloud file storage is a service that stores user files on remote servers (the "cloud") rather than on local devices. Users can access their files from any device with internet connectivity, share files with others, and automatically synchronize changes across all their devices.

**Why it exists:**
- Local storage has limited capacity and is device-bound
- Files need to be accessible from multiple devices
- Collaboration requires shared access to files
- Data needs protection from device loss or failure
- Businesses need centralized file management

**What breaks without it:**
- Files trapped on single devices
- No automatic backup
- Manual file transfer via email or USB
- Version conflicts when collaborating
- No audit trail of changes

---

## 2. Functional Requirements

### 2.1 Core Features (Must Have - P0)

#### File Operations
| Feature | Description | Priority |
|---------|-------------|----------|
| File Upload | Upload files from any device | P0 |
| File Download | Download files to any device | P0 |
| Folder Management | Create, rename, move, delete folders | P0 |
| File Operations | Copy, move, rename, delete files | P0 |
| File Preview | Preview common file types without download | P0 |

#### Synchronization
| Feature | Description | Priority |
|---------|-------------|----------|
| Auto Sync | Automatically sync changes across devices | P0 |
| Selective Sync | Choose which folders to sync locally | P0 |
| Conflict Resolution | Handle simultaneous edits from multiple devices | P0 |
| Offline Access | Access synced files without internet | P0 |

#### Sharing
| Feature | Description | Priority |
|---------|-------------|----------|
| Link Sharing | Generate shareable links for files/folders | P0 |
| User Sharing | Share with specific users/groups | P0 |
| Permission Levels | View, comment, edit permissions | P0 |
| External Sharing | Share with non-registered users | P0 |

### 2.2 Important Features (Should Have - P1)

| Feature | Description | Priority |
|---------|-------------|----------|
| Version History | Access previous versions of files | P1 |
| Search | Full-text search across files | P1 |
| Trash/Recovery | Recover deleted files within retention period | P1 |
| Activity Log | Track all file operations | P1 |
| Notifications | Alert on file changes, shares, comments | P1 |
| Comments | Add comments to files | P1 |
| Starred Items | Mark important files for quick access | P1 |

### 2.3 Nice-to-Have Features (P2)

| Feature | Description | Priority |
|---------|-------------|----------|
| Real-time Collaboration | Multiple users editing simultaneously | P2 |
| File Requests | Request files from others | P2 |
| Watermarking | Add watermarks to shared documents | P2 |
| Expiring Links | Links that expire after time/downloads | P2 |
| Advanced Search | Search by metadata, date, type | P2 |
| Third-party Integrations | Slack, Microsoft Office, etc. | P2 |

---

## 3. Non-Functional Requirements

### 3.1 Performance Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| Upload Throughput | 100 MB/s per user | Large file uploads should complete quickly |
| Download Throughput | 100 MB/s per user | Fast file access |
| Sync Latency | < 5 seconds for small files | Changes should appear quickly on other devices |
| API Response Time | < 200ms (p99) | Responsive UI |
| Search Latency | < 500ms | Quick file discovery |

### 3.2 Scalability Requirements

| Metric | Target | Rationale |
|--------|--------|-----------|
| Total Users | 500 million | Dropbox-scale user base |
| Daily Active Users | 50 million | 10% DAU ratio |
| Concurrent Users | 5 million | 10% of DAU at peak |
| Total Files | 100 billion | Average 200 files per user |
| Total Storage | 500 PB | Average 1 GB per user |
| File Size | Up to 50 GB | Support large video files |

### 3.3 Availability and Reliability

| Metric | Target | Rationale |
|--------|--------|-----------|
| Availability | 99.99% (52 min downtime/year) | Critical for business users |
| Data Durability | 99.999999999% (11 nines) | Files must never be lost |
| Recovery Time Objective | < 1 hour | Quick recovery from failures |
| Recovery Point Objective | 0 (no data loss) | All uploaded data must be preserved |

### 3.4 Security Requirements

| Requirement | Description |
|-------------|-------------|
| Encryption at Rest | AES-256 encryption for all stored files |
| Encryption in Transit | TLS 1.3 for all communications |
| Access Control | Fine-grained permission system |
| Audit Logging | Complete audit trail for compliance |
| Two-Factor Authentication | Optional 2FA for accounts |
| Enterprise SSO | SAML/OIDC integration |

### 3.5 Compliance Requirements

| Standard | Description |
|----------|-------------|
| GDPR | EU data protection compliance |
| HIPAA | Healthcare data handling |
| SOC 2 | Security controls certification |
| ISO 27001 | Information security management |

---

## 4. Clarifying Questions for Interviewers

### Storage and Files

**Q1: What's the maximum file size we need to support?**
> A: 50 GB for video files, but 95% of files are under 100 MB.

**Q2: Do we need to support all file types?**
> A: Yes, but preview only for common types (documents, images, videos, PDFs).

**Q3: How long should we retain deleted files?**
> A: 30 days in trash, 180 days for version history.

### Synchronization

**Q4: How should we handle sync conflicts?**
> A: Keep both versions with conflict suffix, let user resolve. For collaborative docs, last-write-wins with version history.

**Q5: Should sync be real-time or periodic?**
> A: Near real-time (< 5 seconds) for small files, background for large files.

**Q6: Do we need to support selective sync?**
> A: Yes, users should choose which folders sync to each device.

### Sharing

**Q7: What permission levels are needed?**
> A: Viewer (read-only), Commenter (read + comment), Editor (full access), Owner (manage permissions).

**Q8: Can shared links be password protected?**
> A: Yes, with optional password and expiration date.

**Q9: Do we need team/organization features?**
> A: Yes, shared team folders with admin controls.

### Scale and Performance

**Q10: What's our target geography?**
> A: Global, with data residency options for enterprise customers.

**Q11: What's the expected read/write ratio?**
> A: 10:1 (files are read much more often than written).

---

## 5. Success Metrics

### User Engagement Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| DAU/MAU | > 30% | Active user engagement |
| Files Uploaded/Day | 1 billion | Platform usage |
| Sync Success Rate | > 99.9% | Reliability |
| Sharing Rate | 20% of files | Collaboration adoption |

### Performance Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Upload Success Rate | > 99.9% | Reliability |
| Download Success Rate | > 99.99% | Availability |
| Sync Latency (p50) | < 2 seconds | User experience |
| Sync Latency (p99) | < 10 seconds | Tail latency |

### Business Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Storage Cost per GB | < $0.02/month | Operational efficiency |
| Free to Paid Conversion | > 2% | Business model |
| Enterprise Adoption | 50% of revenue | B2B growth |
| Churn Rate | < 2% monthly | Retention |

---

## 6. User Stories

### Personal User Stories

```
As a personal user,
I want to upload photos from my phone,
So that they are backed up and accessible from my computer.

As a personal user,
I want to share a folder with my family,
So that we can all access vacation photos.

As a personal user,
I want to access my files offline,
So that I can work during flights.
```

### Business User Stories

```
As a business user,
I want to share a file with a client via link,
So that they can access it without creating an account.

As a business user,
I want to see who accessed a shared file,
So that I can track document distribution.

As a business user,
I want to recover a previous version of a document,
So that I can undo accidental changes.
```

### Admin User Stories

```
As an IT admin,
I want to set storage quotas for users,
So that I can manage costs.

As an IT admin,
I want to enforce two-factor authentication,
So that company data is secure.

As an IT admin,
I want to view all sharing activity,
So that I can monitor data exposure.
```

---

## 7. Out of Scope

### Explicitly Excluded

| Feature | Reason |
|---------|--------|
| Document Editing | Separate system (Google Docs) |
| Video Streaming | Different architecture needed |
| Email Attachments | Separate email system |
| Backup/Archive | Different product category |
| Desktop App Development | Focus on backend/API |

### Future Considerations

| Feature | Timeline |
|---------|----------|
| AI-powered Search | Phase 2 |
| Smart Organization | Phase 2 |
| Advanced Analytics | Phase 3 |
| Blockchain Verification | Phase 3 |

---

## 8. System Context

### User Types

```
┌─────────────────────────────────────────────────────────────────┐
│                         User Types                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Personal   │  │   Business   │  │  Enterprise  │          │
│  │    Users     │  │    Users     │  │    Admins    │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                 │                 │                   │
│         ▼                 ▼                 ▼                   │
│  • Free tier       • Paid plans      • User management         │
│  • 15 GB storage   • 2 TB storage    • Policy enforcement      │
│  • Basic sharing   • Team folders    • Audit logs              │
│  • 3 devices       • Unlimited       • SSO integration         │
│                      devices                                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Client Types

```
┌─────────────────────────────────────────────────────────────────┐
│                        Client Types                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │  Desktop App │  │  Mobile App  │  │   Web App    │          │
│  │ (Windows,Mac │  │ (iOS,Android)│  │  (Browser)   │          │
│  │   Linux)     │  │              │  │              │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  Features:         Features:         Features:                  │
│  • Full sync       • Photo backup    • File management          │
│  • Offline access  • Offline view    • Sharing                  │
│  • System tray     • Share extension • Preview                  │
│  • Smart sync      • Camera upload   • Search                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 9. Assumptions and Constraints

### Assumptions

1. **Storage**: We can use object storage (S3) for file content
2. **CDN**: Global CDN available for file delivery
3. **Users**: Most users have intermittent connectivity
4. **Files**: Average file size is 5 MB
5. **Usage**: Files are written once, read many times

### Constraints

1. **Budget**: Infrastructure cost must be < $0.02/GB/month
2. **Compliance**: Must support data residency requirements
3. **Legacy**: Must support existing file system conventions
4. **Bandwidth**: Must minimize bandwidth for mobile users
5. **Privacy**: User files must be encrypted and private

---

## 10. Key Challenges

### Technical Challenges

```
1. Efficient Sync Algorithm
   - Detect file changes quickly
   - Minimize data transfer
   - Handle large files
   - Resolve conflicts

2. Storage Efficiency
   - Deduplication across users
   - Compression without quality loss
   - Tiered storage for cost
   - Global replication for durability

3. Scalability
   - Billions of files
   - Millions of concurrent users
   - Petabytes of storage
   - Global distribution

4. Consistency
   - Strong consistency for metadata
   - Eventual consistency for sync
   - Conflict resolution
   - Version management
```

### Business Challenges

```
1. Cost Management
   - Storage costs dominate
   - Bandwidth for sync
   - CDN for downloads
   - Compute for processing

2. Security
   - Zero-knowledge encryption option
   - Enterprise compliance
   - Ransomware protection
   - Data loss prevention

3. User Experience
   - Seamless sync
   - Fast search
   - Intuitive sharing
   - Reliable offline access
```


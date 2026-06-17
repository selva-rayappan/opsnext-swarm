# Domain Events — OpsNext CRM

Domain events represent facts that happened in the system. They are named in past tense, immutable, and carry the data subscribers need to react without querying back.

In v1, events are dispatched synchronously within the NestJS process using an `EventEmitter2` bus. Async event processing (Redis Streams / BullMQ) is the v2 migration path.

---

## Event Envelope (All Events)

Every domain event carries this base structure:

```typescript
interface DomainEvent {
  eventId: string;        // UUID — unique per event instance
  eventType: string;      // e.g., "lead.converted"
  tenantId: string;       // Tenant scope
  actorId: string;        // userId who triggered the event
  occurredAt: DateTime;   // When the event happened
  payload: object;        // Event-specific data (see below)
}
```

---

## IAM Context Events

### UserRegistered

**Trigger**: User completes registration or accepts an invitation.  
**Publisher**: IAM context  
**Subscribers**: Notification (welcome email), Tenant Management (provisioning completion check)

```typescript
payload: {
  userId: string;
  tenantId: string;
  email: string;
  firstName: string;
  lastName: string;
  roleId: string;
  invitedBy?: string;   // userId of inviter, if invited
}
```

---

### UserInvited

**Trigger**: Admin sends a user invitation.  
**Publisher**: IAM  
**Subscribers**: Notification (send invitation email)

```typescript
payload: {
  inviteeEmail: string;
  inviterUserId: string;
  roleId: string;
  invitationToken: string;
  expiresAt: DateTime;
}
```

---

### UserDeactivated

**Trigger**: Admin deactivates a user.  
**Publisher**: IAM  
**Subscribers**: Lead Management (flag unassigned leads), Pipeline (flag orphaned opportunities), Notification (inform admin)

```typescript
payload: {
  userId: string;
  tenantId: string;
  deactivatedBy: string;
  ownedLeadCount: number;
  ownedOpportunityCount: number;
}
```

---

### PasswordResetRequested

**Trigger**: User requests password reset.  
**Publisher**: IAM  
**Subscribers**: Notification (send reset email)

```typescript
payload: {
  userId: string;
  email: string;
  resetToken: string;
  expiresAt: DateTime;
}
```

---

## Tenant Management Events

### TenantProvisioned

**Trigger**: New tenant successfully created with default data.  
**Publisher**: Tenant Management  
**Subscribers**: IAM (create admin user), Notification (send welcome/onboarding email)

```typescript
payload: {
  tenantId: string;
  tenantName: string;
  slug: string;
  plan: string;
  adminUserId: string;
  adminEmail: string;
}
```

---

### TenantSuspended

**Trigger**: Tenant subscription lapses or admin suspends.  
**Publisher**: Tenant Management  
**Subscribers**: IAM (block all logins), Notification (send warning to tenant admin)

```typescript
payload: {
  tenantId: string;
  reason: string;
  suspendedAt: DateTime;
  reactivationDeadline: DateTime;
}
```

---

## Contact & Account Management Events

### AccountCreated

**Trigger**: User creates a new Account.  
**Publisher**: Contact & Account  
**Subscribers**: Analytics (increment account count), Audit Log

```typescript
payload: {
  accountId: string;
  name: string;
  industry?: string;
  ownerId: string;
  source: 'manual' | 'lead_conversion' | 'import';
}
```

---

### ContactCreated

**Trigger**: User creates a new Contact.  
**Publisher**: Contact & Account  
**Subscribers**: Analytics, Audit Log

```typescript
payload: {
  contactId: string;
  email?: string;
  accountId?: string;
  ownerId: string;
  source: 'manual' | 'lead_conversion' | 'import';
}
```

---

### ContactsMerged

**Trigger**: Admin merges two duplicate Contact records.  
**Publisher**: Contact & Account  
**Subscribers**: Activity (re-parent activities), Lead (re-parent leads), Pipeline (re-parent opportunities), Audit Log

```typescript
payload: {
  survivingContactId: string;
  mergedContactId: string;
  mergedBy: string;
  transferredActivityCount: number;
  transferredLeadCount: number;
  transferredOpportunityCount: number;
}
```

---

## Lead Management Events

### LeadCreated

**Trigger**: Lead created manually, via import, or via API.  
**Publisher**: Lead Management  
**Subscribers**: Notification (if assigned on creation), Analytics, Audit Log

```typescript
payload: {
  leadId: string;
  firstName: string;
  lastName: string;
  email?: string;
  source: string;
  ownerId?: string;
  teamId?: string;
  createdBy: string;
}
```

---

### LeadAssigned

**Trigger**: Lead ownership changed (manual assignment or round-robin).  
**Publisher**: Lead Management  
**Subscribers**: Notification (notify new owner), Activity (log assignment change), Audit Log

```typescript
payload: {
  leadId: string;
  previousOwnerId?: string;
  newOwnerId: string;
  assignedBy: string;
  assignmentMethod: 'manual' | 'round_robin';
}
```

---

### LeadQualified

**Trigger**: Lead status changed to `qualified`.  
**Publisher**: Lead Management  
**Subscribers**: Analytics, Notification (optional — manager alert), Audit Log

```typescript
payload: {
  leadId: string;
  qualifiedBy: string;
  qualificationNotes?: string;
}
```

---

### LeadDisqualified

**Trigger**: Lead marked as disqualified with a reason.  
**Publisher**: Lead Management  
**Subscribers**: Analytics (track disqualification reasons), Audit Log

```typescript
payload: {
  leadId: string;
  disqualifiedBy: string;
  reason: string;
  notes?: string;
}
```

---

### LeadConverted

**Trigger**: Qualified lead converted to Contact/Account/Opportunity.  
**Publisher**: Lead Management  
**Subscribers**: Analytics (lead conversion rate), Notification (confirm conversion to owner), Audit Log

```typescript
payload: {
  leadId: string;
  convertedBy: string;
  contactId?: string;
  accountId?: string;
  opportunityId?: string;
  convertedAt: DateTime;
}
```

---

## Sales Pipeline Events

### OpportunityCreated

**Trigger**: Opportunity created manually or from lead conversion.  
**Publisher**: Sales Pipeline  
**Subscribers**: Analytics, Notification (if assigned to someone else), Audit Log

```typescript
payload: {
  opportunityId: string;
  name: string;
  pipelineId: string;
  stageId: string;
  amount?: number;
  currency?: string;
  expectedCloseDate?: string;
  ownerId: string;
  accountId?: string;
  source: 'manual' | 'lead_conversion';
}
```

---

### OpportunityStageChanged

**Trigger**: Opportunity moved to a different stage (drag-drop or API).  
**Publisher**: Sales Pipeline  
**Subscribers**: Activity (log stage change in timeline), Analytics (stage velocity), Notification (manager alert on close), Audit Log

```typescript
payload: {
  opportunityId: string;
  name: string;
  pipelineId: string;
  previousStageId: string;
  previousStageName: string;
  newStageId: string;
  newStageName: string;
  movedBy: string;
  amount?: number;
}
```

---

### OpportunityClosedWon

**Trigger**: Opportunity stage set to a terminal Won stage.  
**Publisher**: Sales Pipeline  
**Subscribers**: Analytics (update win rate, revenue metrics), Notification (congratulate owner + alert manager), Audit Log

```typescript
payload: {
  opportunityId: string;
  name: string;
  amount: number;
  currency: string;
  closedBy: string;
  accountId?: string;
  closedAt: DateTime;
}
```

---

### OpportunityClosedLost

**Trigger**: Opportunity stage set to a terminal Lost stage.  
**Publisher**: Sales Pipeline  
**Subscribers**: Analytics (update win/loss rate, track loss reasons), Notification (alert manager), Audit Log

```typescript
payload: {
  opportunityId: string;
  name: string;
  amount?: number;
  lostReason: string;
  closedBy: string;
  accountId?: string;
  closedAt: DateTime;
}
```

---

### OpportunityOwnerChanged

**Trigger**: Opportunity reassigned to a different owner.  
**Publisher**: Sales Pipeline  
**Subscribers**: Activity (log ownership change), Notification (notify new owner), Audit Log

```typescript
payload: {
  opportunityId: string;
  previousOwnerId: string;
  newOwnerId: string;
  changedBy: string;
}
```

---

## Activity & Task Events

### TaskCreated

**Trigger**: Task created by a user.  
**Publisher**: Activity & Task  
**Subscribers**: Notification (if assigned to someone else), Audit Log

```typescript
payload: {
  taskId: string;
  title: string;
  assigneeId: string;
  creatorId: string;
  dueDate?: DateTime;
  priority: string;
  entityType?: string;
  entityId?: string;
}
```

---

### TaskDue

**Trigger**: A scheduled job fires when a task's dueDate is reached and status is still `open`.  
**Publisher**: Activity & Task (scheduled job)  
**Subscribers**: Notification (remind assignee)

```typescript
payload: {
  taskId: string;
  title: string;
  assigneeId: string;
  dueDate: DateTime;
  entityType?: string;
  entityId?: string;
}
```

---

### TaskCompleted

**Trigger**: Assignee or creator marks a task as complete.  
**Publisher**: Activity & Task  
**Subscribers**: Notification (notify creator if different from assignee), Analytics, Audit Log

```typescript
payload: {
  taskId: string;
  completedBy: string;
  completedAt: DateTime;
}
```

---

### ActivityLogged

**Trigger**: Any activity (call, meeting, note) is logged.  
**Publisher**: Activity & Task  
**Subscribers**: Analytics (activity cadence metrics), (no notification in v1)

```typescript
payload: {
  activityId: string;
  type: 'call' | 'meeting' | 'note';
  entityType: string;
  entityId: string;
  loggedBy: string;
  occurredAt: DateTime;
}
```

---

## Communication Events

### EmailLogged

**Trigger**: Email manually logged or captured via BCC.  
**Publisher**: Email & Communication  
**Subscribers**: Activity (adds email to entity timeline), Analytics, Audit Log

```typescript
payload: {
  emailId: string;
  direction: 'inbound' | 'outbound';
  contactId?: string;
  accountId?: string;
  leadId?: string;
  opportunityId?: string;
  loggedBy: string;
  sentAt: DateTime;
}
```

---

## Notification Events (Internal)

These events are consumed exclusively by the Notification Delivery context.

### NotificationQueued

**Trigger**: Any upstream event that requires user notification.  
**Publisher**: Notification Delivery (internal)  
**Subscribers**: Email delivery (SMTP), in-app notification store

```typescript
payload: {
  notificationId: string;
  recipientUserId: string;
  type: string;
  title: string;
  body: string;
  channels: ('in_app' | 'email')[];
  entityType?: string;
  entityId?: string;
}
```

---

## Event Catalog Summary

| Event | Context | Triggers | Key Subscribers |
|-------|---------|----------|----------------|
| UserRegistered | IAM | Registration/invite accept | Notification |
| UserInvited | IAM | Admin invite | Notification |
| UserDeactivated | IAM | Admin deactivate | Lead, Pipeline, Notification |
| PasswordResetRequested | IAM | Forgot password | Notification |
| TenantProvisioned | Tenant | New tenant signup | IAM, Notification |
| TenantSuspended | Tenant | Billing lapse | IAM, Notification |
| AccountCreated | Contact & Account | Account create | Analytics, Audit |
| ContactCreated | Contact & Account | Contact create | Analytics, Audit |
| ContactsMerged | Contact & Account | Merge operation | Activity, Lead, Pipeline, Audit |
| LeadCreated | Lead | Lead create | Notification, Analytics, Audit |
| LeadAssigned | Lead | Assignment | Notification, Activity, Audit |
| LeadQualified | Lead | Qualification | Analytics, Audit |
| LeadDisqualified | Lead | Disqualification | Analytics, Audit |
| LeadConverted | Lead | Conversion | Analytics, Notification, Audit |
| OpportunityCreated | Pipeline | Opp create | Analytics, Notification, Audit |
| OpportunityStageChanged | Pipeline | Stage change | Activity, Analytics, Notification, Audit |
| OpportunityClosedWon | Pipeline | Won close | Analytics, Notification, Audit |
| OpportunityClosedLost | Pipeline | Lost close | Analytics, Notification, Audit |
| OpportunityOwnerChanged | Pipeline | Reassign | Activity, Notification, Audit |
| TaskCreated | Activity | Task create | Notification, Audit |
| TaskDue | Activity (scheduled) | Due date reached | Notification |
| TaskCompleted | Activity | Task complete | Notification, Analytics, Audit |
| ActivityLogged | Activity | Call/meeting/note | Analytics |
| EmailLogged | Communication | Email capture | Activity, Analytics, Audit |

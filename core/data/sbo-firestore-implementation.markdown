# SBO Firestore Implementation

This document describes the Firestore structure and connection strategies for a SaaS tool providing digital solutions to businesses. The structure uses two main collections: `sbo_accounts` for small business owners and authorized personnel, and `sbo_connections` for other entities, with business entities stored in `sbo_connections/entities` parallel to `sbo_connections/identities`. It includes strategies to ensure efficient and proper connections between records.

## Firestore Structure
The Firestore structure uses subcollections to handle relationships, reducing data duplication and improving query performance.

```
sbo_accounts (collection)
├── {accountId} (document: identity of a small business owner or authorized personnel)
│   ├── email: string
│   ├── firstName: string
│   ├── lastName: string
│   ├── phoneNumber: string
│   ├── role: array<string>
│   ├── createdAt: timestamp
│   ├── updatedAt: timestamp
│   ├── businesses (subcollection)
│   │   ├── {businessId} (document)
│   │   │   ├── businessId: string (reference to sbo_connections/entities)
│   │   │   ├── roleInBusiness: string
│   ├── linkedIdentities (subcollection)
│   │   ├── {identityId} (document)
│   │   │   ├── identityId: string (reference to sbo_accounts or sbo_connections/identities)
│   │   │   ├── relationshipType: string
│
sbo_connections (collection)
├── identities (subcollection)
│   ├── {identityId} (document: non-owner identities, e.g., employees, customers)
│   │   ├── email: string
│   │   ├── firstName: string
│   │   ├── lastName: string
│   │   ├── phoneNumber: string
│   │   ├── role: array<string>
│   │   ├── createdAt: timestamp
│   │   ├── updatedAt: timestamp
│   │   ├── businesses (subcollection)
│   │   │   ├── {businessId} (document)
│   │   │   │   ├── businessId: string (reference to sbo_connections/entities)
│   │   │   │   ├── roleInBusiness: string
│   │   ├── linkedIdentities (subcollection)
│   │   │   ├── {identityId} (document)
│   │   │   │   ├── identityId: string
│   │   │   │   ├── relationshipType: string
├── entities (subcollection)
│   ├── {businessId} (document: business entities, e.g., owned businesses, contractors, suppliers)
│   │   ├── name: string
│   │   ├── ownerId: string (reference to sbo_accounts)
│   │   ├── type: string
│   │   ├── industry: string
│   │   ├── address: object
│   │   ├── contactEmail: string
│   │   ├── contactPhone: string
│   │   ├── website: string
│   │   ├── socialAccounts: array<object>
│   │   ├── hours: object
│   │   ├── tags: array<string>
│   │   ├── mainProducts: array<object>
│   │   ├── mainServices: array<object>
│   │   ├── summary: string
│   │   ├── description: string
│   │   ├── legalName: string
│   │   ├── taxId: string
│   │   ├── foundedDate: date
│   │   ├── employeeCount: number
│   │   ├── certifications: array<string>
│   │   ├── createdAt: timestamp
│   │   ├── updatedAt: timestamp
│   │   ├── associatedIdentities (subcollection)
│   │   │   ├── {identityId} (document)
│   │   │   │   ├── identityId: string
│   │   │   │   ├── role: string
│   │   ├── associatedBusinesses (subcollection)
│   │   │   ├── {businessId} (document)
│   │   │   │   ├── businessId: string (reference to sbo_connections/entities)
│   │   │   │   ├── relationshipType: string
```

## Ensuring Proper Connections
To manage relationships efficiently in Firestore and your application, follow these strategies:

1. **Reference-Based Relationships**:
   - Use document references (e.g., `businessId`, `identityId`) to link records across `sbo_accounts` and `sbo_connections`. For example, an identity’s `businesses` subcollection references business documents in `sbo_connections/entities`.
   - Store reciprocal references to maintain bidirectional relationships. For instance, if an identity is linked to a business, the business’s `associatedIdentities` subcollection includes the identity, and the identity’s `businesses` subcollection includes the business.

2. **Subcollections for Scalability**:
   - Subcollections (`businesses`, `linkedIdentities`, `associatedIdentities`, `associatedBusinesses`) allow efficient querying of relationships without duplicating data. For example, to find all businesses an identity owns, query the `businesses` subcollection under their `sbo_accounts/{accountId}` document.
   - Firestore’s subcollection queries are scoped to the parent document, reducing the need for complex joins.

3. **Role-Based Access Control**:
   - Use the `role` field in identities and `roleInBusiness` in subcollections to enforce permissions. For example, only identities with `role: ["owner", "authorized_personnel"]` in `sbo_accounts` can create or edit businesses.
   - Implement Firestore Security Rules to restrict access. Example:
     ```javascript
     rules_version = '2';
     service cloud.firestore {
       match /databases/{database}/documents {
         match /sbo_accounts/{accountId} {
           allow read, write: if request.auth != null && request.auth.uid == accountId;
           match /businesses/{businessId} {
             allow read, write: if request.auth != null && request.auth.uid == accountId;
           }
         }
         match /sbo_connections/identities/{identityId} {
           allow read: if request.auth != null;
           allow write: if request.auth != null && exists(/databases/$(database)/documents/sbo_accounts/$(request.auth.uid));
         }
         match /sbo_connections/entities/{businessId} {
           allow read: if request.auth != null;
           allow write: if request.auth != null && exists(/databases/$(database)/documents/sbo_accounts/$(request.auth.uid));
         }
       }
     }
     ```

4. **Application Logic for Connections**:
   - **Creating Relationships**: When linking an identity to a business, update both the identity’s `businesses` subcollection and the business’s `associatedIdentities` subcollection in a single Firestore batch write to ensure consistency. Example:
     ```javascript
     const db = firebase.firestore();
     async function addEmployeeToBusiness(accountId, businessId, employeeEmail, role) {
       const batch = db.batch();
       const employeeRef = db.collection('sbo_connections').doc('identities').collection('identities').doc();
       batch.set(employeeRef, {
         email: employeeEmail,
         firstName: '',
         lastName: '',
         role: ['employee'],
         createdAt: firebase.firestore.FieldValue.serverTimestamp(),
         updatedAt: firebase.firestore.FieldValue.serverTimestamp(),
       });
       const employeeBusinessRef = employeeRef.collection('businesses').doc(businessId);
       batch.set(employeeBusinessRef, { businessId, roleInBusiness: role });
       const businessRef = db.collection('sbo_connections').doc('entities').collection('entities').doc(businessId);
       const businessEmployeeRef = businessRef.collection('associatedIdentities').doc(employeeRef.id);
       batch.set(businessEmployeeRef, { identityId: employeeRef.id, role });
       await batch.commit();
     }
     ```
   - **Querying Relationships**: Use Firestore queries to fetch related data. For example, to get all employees of a business:
     ```javascript
     db.collection('sbo_connections')
       .doc('entities')
       .collection('entities')
       .doc(businessId)
       .collection('associatedIdentities')
       .where('role', '==', 'employee')
       .get();
     ```
   - **Cascading Updates**: If a business or identity is deleted, use Cloud Functions to clean up references in related subcollections to prevent orphaned data.

5. **Indexing for Performance**:
   - Create composite indexes for frequent queries, such as filtering identities by `role` or businesses by `industry` or `tags`. Define these in the `firestore.indexes.json` file:
     ```json
     {
       "indexes": [
         {
           "collectionGroup": "associatedIdentities",
           "queryScope": "COLLECTION",
           "fields": [
             { "fieldPath": "role", "order": "ASCENDING" },
             { "fieldPath": "identityId", "order": "ASCENDING" }
           ]
         },
         {
           "collectionGroup": "entities",
           "queryScope": "COLLECTION",
           "fields": [
             { "fieldPath": "industry", "order": "ASCENDING" },
             { "fieldPath": "businessId", "order": "ASCENDING" }
           ]
         },
         {
           "collectionGroup": "entities",
           "queryScope": "COLLECTION",
           "fields": [
             { "fieldPath": "tags", "arrayConfig": "CONTAINS" }
           ]
         }
       ]
     }
     ```
   - Deploy indexes using `firebase deploy --only firestore:indexes`.

6. **Denormalization for Efficiency**:
   - Store minimal redundant data (e.g., `name` in the `businesses` subcollection of an identity) to reduce the need for multiple queries, but keep references as the primary link.
   - Use Firestore’s `FieldValue.arrayUnion` and `FieldValue.arrayRemove` to manage arrays like `role`, `tags`, or `socialAccounts` without overwriting other values.

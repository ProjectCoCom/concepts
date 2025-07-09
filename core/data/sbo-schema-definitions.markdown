# SBO Schema Definitions for Firestore

This document outlines the schema definitions for a SaaS tool providing digital solutions to businesses, implemented in Firestore. The schemas support complex relationships where a single identity (owner or authorized personnel) can host multiple businesses and identities (partners), and a business can host multiple identities (employees, customers, clients) and other businesses (contractors, partners, suppliers). The main collections are `sbo_accounts` for small business owners and authorized personnel accounts, and `sbo_connections` for all other entities, with business entities stored in `sbo_connections/entities`.

## Identity/Profile Schema
The identity schema represents individuals (owners, authorized personnel, partners, employees, customers, etc.) and captures personal and role-related details while supporting relationships with businesses and other identities.

- **id**: Unique identifier (Firestore document ID, auto-generated).
- **email**: Primary contact email (string, indexed for queries).
- **firstName**: First name (string).
- **lastName**: Last name (string).
- **phoneNumber**: Contact phone number (string, optional).
- **role**: Role(s) associated with the identity (array of strings, e.g., ["owner", "authorized_personnel", "employee", "customer"]).
- **createdAt**: Timestamp of account creation (timestamp).
- **updatedAt**: Timestamp of last update (timestamp).
- **businesses**: Subcollection of businesses the identity is associated with.
  - **businessId**: Reference to a business document in `sbo_connections/entities` (string).
  - **roleInBusiness**: Role within the business (string, e.g., "owner", "employee", "customer").
- **linkedIdentities**: Subcollection of other identities this identity is connected to (e.g., partners).
  - **identityId**: Reference to another identity in `sbo_accounts` or `sbo_connections/identities` (string).
  - **relationshipType**: Type of relationship (string, e.g., "partner", "collaborator").

## Business Information Schema
The business schema represents business entities (owned businesses, contractors, suppliers, etc.) and includes comprehensive operational and descriptive details, as well as relationships with identities and other businesses. Business entities are stored in the `sbo_connections/entities` subcollection.

- **id**: Unique identifier (Firestore document ID, auto-generated).
- **name**: Business name (string).
- **ownerId**: Reference to the primary owner’s identity in `sbo_accounts` (string).
- **type**: Business type (string, e.g., "retail", "service", "contractor").
- **industry**: Industry category (string, e.g., "technology", "hospitality").
- **address**: Physical address (object: { street, city, state, zip, country }).
- **contactEmail**: Business contact email (string).
- **contactPhone**: Business contact phone (string, optional).
- **website**: Business website URL (string, optional).
- **socialAccounts**: Array of social media profiles (array of objects: { platform: string, handle: string, url: string }, e.g., [{ platform: "Twitter", handle: "@example", url: "https://twitter.com/example" }]).
- **hours**: Business operating hours (object: { monday: { open: string, close: string }, ..., sunday: { open: string, close: string } }, e.g., { monday: { open: "09:00", close: "17:00" } }).
- **tags**: Keywords describing the business (array of strings, e.g., ["eco-friendly", "local", "family-owned"]).
- **mainProducts**: Primary products offered (array of objects: { name: string, description: string, category: string, priceRange: string }, e.g., [{ name: "Organic Coffee", description: "Locally sourced beans", category: "Beverages", priceRange: "$10-$20" }]).
- **mainServices**: Primary services offered (array of objects: { name: string, description: string, category: string, priceRange: string }, e.g., [{ name: "Web Design", description: "Custom website development", category: "Digital Services", priceRange: "$500-$2000" }]).
- **summary**: Brief business overview (string, max 200 characters, e.g., "Local coffee shop offering organic blends and cozy ambiance").
- **description**: Detailed business description (string, max 1000 characters, e.g., "Our coffee shop sources organic beans from local farmers...").
- **legalName**: Legal business name (string, optional, e.g., "Example Coffee Co. LLC").
- **taxId**: Tax identification number (string, optional, encrypted if stored).
- **foundedDate**: Date the business was founded (date, optional, e.g., "2020-01-01").
- **employeeCount**: Approximate number of employees (number, optional, e.g., 10).
- **certifications**: Industry certifications or licenses (array of strings, e.g., ["ISO 9001", "Organic Certified"]).
- **createdAt**: Timestamp of business creation (timestamp).
- **updatedAt**: Timestamp of last update (timestamp).
- **associatedIdentities**: Subcollection of identities linked to the business (e.g., employees, customers).
  - **identityId**: Reference to an identity in `sbo_accounts` or `sbo_connections/identities` (string).
  - **role**: Role of the identity in the business (string, e.g., "employee", "customer").
- **associatedBusinesses**: Subcollection of other businesses linked to this business (e.g., contractors, suppliers).
  - **businessId**: Reference to another business in `sbo_connections/entities` (string).
  - **relationshipType**: Type of relationship (string, e.g., "contractor", "supplier").

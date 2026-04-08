# StockFlow Case Study Solution

This repository contains my submission for the StockFlow B2B Inventory Management System assignment.

## Contents

- [Part 1: Code Review & Debugging](STOCKFLOW_SOLUTION.md#part-1-code-review--debugging)
- [Part 2: Database Design (MongoDB)](STOCKFLOW_SOLUTION.md#part-2-database-design-mongodb)
- [Part 3: API Implementation - Low Stock Alerts](STOCKFLOW_SOLUTION.md#part-3-api-implementation---low-stock-alerts)
- [Assumptions Summary](STOCKFLOW_SOLUTION.md#summary-of-assumptions)

## Main Submission File

- [STOCKFLOW_SOLUTION.md](STOCKFLOW_SOLUTION.md)

## What This Submission Includes

1. Identification of technical and business-logic issues in the provided product creation endpoint.
2. Production impact analysis for each issue.
3. Corrected API implementation with validation, transaction handling, and safer error responses.
4. MongoDB schema design for multi-warehouse inventory, suppliers, bundles, and inventory movement history.
5. Missing-requirements questions for product and stakeholder clarification.
6. Low-stock alerts endpoint implementation with supplier enrichment and stockout estimation logic.
7. Edge-case handling and sample API request/response.

## Technical Stack Used

- Node.js
- Express.js
- MongoDB + Mongoose
- Joi validation

## Notes for Reviewers

- The solution is intentionally explicit to make design assumptions and trade-offs easy to discuss in a live review.
- Business rules that were ambiguous in the prompt are documented in the assumptions section.
- API and schema choices are optimized for maintainability, clarity, and realistic production behavior.

## Author

Case-study submission by candidate.

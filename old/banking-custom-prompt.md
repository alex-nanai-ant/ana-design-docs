As we interact with the user, augment any documents available in the internal documents repository with information from the Client organization’s websites and the additional information sources identified here.

Client Organization websites:
- https://www.bibank.com/

Additional information sources are as follows:

## Regulatory Sources

### 1. Interagency & Primary Banking Regulators (U.S.)

- **FFIEC IT Examination Manuals:** Baseline standards for IT security, business continuity, vendor risk, and operational resilience.
  - **URL:** https://ithandbook.ffiec.gov/
- **CFPB Interactive Regulations & Compliance Guides:** Plain-language guides and consumer compliance rulebooks (Regulations E, Z, B, CC, etc.).
  - **Interactive Regulations:** https://www.consumerfinance.gov/rules-policy/regulations/
  - **eCFR Title 12 Chapter X:** https://www.ecfr.gov/current/title-12/chapter-X
- **OCC Comptroller’s Handbooks & Bulletins:** Operational risk management guidance covering credit, liquidity, retail lending, and governance.
  - **URL:** https://www.occ.gov/publications-and-resources/publications/comptrollers-handbook/index-comptrollers-handbook.html
- **FDIC Consumer Compliance Examination Manual:** Detailed manuals and compliance checklists for consumer protection, deposit insurance, and fair lending.
  - **URL:** https://www.fdic.gov/resources/bankers/consumer-compliance-examination-manual/index.html

### 2. Financial Crime, AML, & Sanctions Guidance

- **FinCEN Advisories & Guidance:** Suspicious Activity Report (SAR) filing guidance, Currency Transaction Reporting (CTR) thresholds, and red flags.
  - **Guidance Directory:** https://www.fincen.gov/resources/statutes-regulations/guidance
  - **Advisories:** https://www.fincen.gov/resources/advisoriesbulletinsfact-sheets/advisories
- **OFAC Sanctions & Compliance Guidance:** Official interpretations on sanctions screening, SDN lists, and country-specific programs.
  - **URL:** https://ofac.treasury.gov/sanctions-compliance-and-evaluation
- **FFIEC BSA/AML Examination Manual:** The master manual used by all U.S. banking agencies to evaluate AML compliance programs.
  - **URL:** https://bsaaml.ffiec.gov/manual/

### 3. Cybersecurity & Operational Risk Frameworks

- **NIST Cybersecurity Framework (CSF):** Core framework for access control, incident handling, and risk assessment.
  - **URL:** https://www.nist.gov/cyberframework
- **Center for Internet Security (CIS) Controls:** Practical, prioritized safeguards for enterprise networks and end-user devices.
  - **URL:** https://www.cisecurity.org/controls

### 4. Industry Associations & Best Practice Repositories

- **American Bankers Association (ABA):** Industry whitepapers, training materials, and operational reference guides.
  - **URL:** https://www.aba.com/news-research
- **Risk Management Association (RMA):** Frameworks for credit risk, enterprise risk management (ERM), and market risk.
  - **URL:** https://www.rmahq.org/

## Beyond Regulations

Beyond primary banking regulators and trade associations, banks rely on operational networks, technology vendors, legal repositories, and consumer guidance sites to build out detailed SOPs. These sources contain the practical "how-to" details—such as specific network rules, code tables, and customer-facing explanations—that statutory regulations do not cover.

### 1. Payment Networks & Clearinghouses (Operational & Exception Rules)

*Use these to build SOPs for transaction processing timelines, chargebacks, check clearing, and wire routing.*

- **NCHA (The ACH Association / Nacha):** Operating Rules & Guidelines governing ACH transfers, unauthorized return reason codes (R01–R99), and RDFI/ODFI responsibilities.
  - **URL:** https://www.nacha.org/rules
- **Visa Product and Service Rules & Chargeback Guides:** Technical rules for card dispute processing, representment timelines, merchant category codes, and fraud liability shifts.
  - **URL:** https://usa.visa.com/support/merchant/merchant-agreements.html
- **Mastercard Rules & Chargeback Management Manual:** Standard operating guidelines for processing Mastercard debit/credit disputes, card reissuance, and digital wallet tokenization.
  - **URL:** https://www.mastercard.us/en-us/business/overview/support/rules.html
- **The Clearing House (TCH) & Real-Time Payments (RTP):** Rules for wire transfers, CHIPS processing, and RTP instant payment settlement.
  - **URL:** https://www.theclearinghouse.org/payment-systems/rtp/rules
- **Federal Reserve Financial Services (FRFS):** Operating Circulars (e.g., OC 3 for check collection, OC 5 for Fedwire, OC 8 for FedACH).
  - **URL:** https://www.frbservices.org/resources/rules-regulations/operating-circulars.html

### 2. Core Banking Vendors & Software Knowledge Bases (System Click-Paths)

*Use these to document system-level "how-to" steps for tellers and CSRs (e.g., screen names, field validation, and override codes).*

- **Fiserv Product Documentation (Premier, Precision, Signature):** Guides for executing account freezes, fee waivers, deposit hold flags, and ledger adjustments.
  - **URL:** Access via client portal (https://www.fiserv.com/en/client-portal.html)
- **FIS Global Documentation (Horizon, Systematics, FIS Digital):** Core banking operational workflows for account maintenance, wire generation, and loan servicing.
  - **URL:** Access via FIS Client Connection Portal (https://www.fisglobal.com/en/client-login)
- **Jack Henry & Associates (SilverLake, CIF 20/20, Symitar):** Operational knowledge bases for credit unions and community banks handling daily branch transactions.
  - **URL:** Access via ForCustomer Portal (https://www.jackhenry.com/)
- **Q2 / Alkami Online Banking Portals:** Administrator guides for setting up business online banking users, multi-factor authentication (MFA) resets, and ACH limit rules.

### 3. Legal, Statutory Code & State Regulatory Repositories

*Use these to establish precise legal definitions for estate handling, power of attorney, tax levies, and state-level disclosures.*

- **Uniform Commercial Code (UCC) via Cornell Legal Information Institute:**
  - **UCC Article 3 (Negotiable Instruments):** Foundation for check endorsements, holder-in-due-course rules, and altered check liabilities.
  - **UCC Article 4 (Bank Deposits & Collections):** Customer-bank relationship rules and duty to examine statements.
  - **UCC Article 4A (Funds Transfers):** Commercial wire transfer liabilities and security procedure standards.
  - **URL:** https://www.law.cornell.edu/ucc
- **Electronic Code of Federal Regulations (eCFR):** Dynamic, updated versions of federal banking laws (Title 12 - Banks and Banking).
  - **URL:** https://www.ecfr.gov/current/title-12
- **State Bankers Associations (e.g., California Bankers, Texas Bankers):** State-specific legal manuals covering power of attorney (POA), probate/deceased account handling, and state garnishment/levy order priorities.

### 4. Consumer-Facing Educational & Plain-Language Repositories

*Use these to curate customer-facing FAQ content and train the chatbot to generate clear, non-jargon explanations.*

- **OCC HelpWithMyBank.gov:** Official answers to over 250 common consumer banking questions regarding holds, disputes, fees, and account closures.
  - **URL:** https://www.helpwithmybank.gov/
- **CFPB Consumer Education & Answers:** Plain-language guides explaining mortgage disclosures, debit card dispute rights, and Zelle/scam liabilities.
  - **URL:** https://www.consumerfinance.gov/consumer-tools/bank-accounts/
- **FTC Consumer Advice (Financial Scams & Privacy):** Guidance on identifying fake check scams, identity theft recovery steps, and imposter fraud.
  - **URL:** https://consumer.ftc.gov/identity-theft-and-online-security
# Commercial End User License Agreement (EULA)

**IMPORTANT: PLEASE READ THIS COMMERCIAL END USER LICENSE AGREEMENT ("AGREEMENT") CAREFULLY BEFORE DOWNLOADING, INSTALLING, COPYING, OR USING CRENOX PRO SOFTWARE. BY INSTALLING, EXECUTING, OR ACCESSING CRENOX PRO, YOU ("CUSTOMER" OR "LICENSEE") AGREE TO BE BOUND BY ALL TERMS AND CONDITIONS OF THIS AGREEMENT. IF YOU DO NOT AGREE TO THESE TERMS, DO NOT INSTALL, ACCESS, OR USE THE SOFTWARE.**

This Agreement is a binding legal contract entered into by and between **CrenoxHQ** ("Licensor") and the corporate entity, government agency, or organization purchasing or evaluating the Software ("Customer").

---

### Section 1. Definitions

1. **"Authorized Seat"** means a distinct physical workstation, virtual desktop instance, or continuous integration (CI/CD) runner actively enrolled to execute the Software agent within the seat quota specified in Customer's License File.
2. **"Beta Software"** means any version of the Software designated as "beta", "pilot", "preview", or bearing a release version prior to `1.0.0`.
3. **"Confidential Information"** means any non-public technical, operational, or commercial information disclosed by one party to the other, including Software binaries, License Files, pricing terms, security architecture, and Customer diagnostic reports.
4. **"Documentation"** means the official installation, administration, and technical reference guides provided by Licensor with the Software.
5. **"License File"** means the digitally signed activation file (`crenox.lic`) issued by Licensor specifying Customer's legal identity, Authorized Seat quota, licensed feature entitlements, designated Node Fingerprint, and subscription term.
6. **"Master Node"** means the central management server software deployed on Customer's infrastructure for policy distribution, seat administration, exemption approval, and audit logging.
7. **"Node Fingerprint"** means the cryptographic hardware host identifier generated on the server executing the Master Node.
8. **"Order Schedule"** means a mutually executed purchase order, quote, or commercial invoice referencing this Agreement that specifies the subscription tier, seat count, subscription term, and applicable fees.
9. **"Software"** means the compiled Crenox Pro server and agent executables, embedded web console, built-in detection signature catalog, installation scripts, and official updates provided by Licensor.

---

### Section 2. License Grant, Seat Reallocation & Server Migration

1. **Grant of License**: Subject to the terms and conditions of this Agreement and timely payment of all applicable fees, Licensor grants Customer a non-exclusive, non-transferable, non-sublicensable, limited license during the active subscription term to:
   - Install and execute one (1) production instance of the Master Node bound to Customer's designated Node Fingerprint within Customer's private network;
   - Install and execute the workstation agent across Customer's workstations and CI/CD runners up to the aggregate number of active Authorized Seats specified in the License File; and
   - Execute pre-commit scans, historical repository scans, and history redactions on codebases owned or controlled by Customer (including use by Customer's authorized contractors acting solely on Customer's behalf).
2. **Seat Reallocation**: Customer may revoke or delete an Authorized Seat assigned to a decommissioned machine, departed user, or terminated CI worker and reassign that seat to a replacement workstation without additional fee.
3. **Backup & Archival Copies**: Customer may maintain cold-standby and archival copies of the Software binaries, databases, and License Files solely for disaster recovery and business continuity purposes, provided that no more than one (1) production Master Node instance is actively operating at any given time, except during a migration period permitted under Section 2.6.
4. **Evaluation Licenses**: If Customer is issued an evaluation or trial License File, Customer may use the Software solely for internal non-production evaluation during the trial period specified in the License File (default: thirty (30) calendar days). Evaluation licenses terminate automatically upon expiration of the trial period unless converted to a paid subscription.
5. **Beta Software**: Beta Software is licensed solely on an "AS IS" and "AS AVAILABLE" basis without service level commitments or warranties of any kind. Any feedback, suggestions, or bug reports voluntarily submitted by Customer may be used and incorporated by Licensor into the Software without restriction or compensation.
6. **Server Migration & License Re-Issuance**:
   - **License Re-Issuance**: If Customer replaces or migrates the physical or virtual host executing the Master Node, Customer may submit the new host's Node Fingerprint to Licensor together with written confirmation of decommission of the prior host. Licensor shall issue a replacement License File bound to the new Node Fingerprint at no additional fee, up to two (2) times per twelve (12) month subscription term.
   - **14-Day Migration Grace Period**: When the Master Node detects a change in the underlying host Node Fingerprint, the Software provides a fourteen (14) calendar day migration grace period during which Customer may transition from the retiring host to the replacement host. Customer shall decommission the retired instance upon completion of cutover and in no event later than the expiration of the fourteen (14) calendar day period.

---

### Section 3. Fees, Payment Terms & Taxes

1. **Orders & Invoicing**: Subscription fees are governed by the applicable Order Schedule referencing this Agreement. Unless otherwise specified in the Order Schedule, subscription fees are billed annually in advance and are non-refundable except as expressly set forth in Sections 6.3 and 11.1.
2. **Payment Terms**: Unless otherwise stated in the Order Schedule, invoices are payable within thirty (30) calendar days of the invoice date (Net 30) for approved enterprise purchase orders, or prior to issuance of the production License File for initial orders.
3. **Payment Methods**: Payments shall be settled via one of the following methods specified on the invoice:
   - **United States Dollar (USD)**: ACH or Fedwire bank transfer;
   - **Euro (EUR)**: SEPA or SEPA Instant credit transfer;
   - **British Pound (GBP)**: Faster Payments Service (FPS) or BACS transfer; or
   - **Digital Assets**: USDC or USDT transfer to Licensor's designated corporate address.
4. **Taxes & Transfer Charges**: All fees are exclusive of taxes. Customer is responsible for all applicable sales, use, value-added (VAT), goods and services (GST), or withholding taxes, excluding taxes based on Licensor's net income. Each party shall bear its own banking or network transfer fees.
5. **Term Price Stability**: The per-seat rate specified in an active Order Schedule shall remain fixed for the duration of that initial or renewal subscription term. Any fee adjustments shall apply only to subsequent renewal terms upon prior written notice before the renewal date.

---

### Section 4. On-Premises Operation, Data Ownership & Customer Infrastructure Security

1. **Local Execution**: The Software is designed to execute within Customer's self-hosted infrastructure. The Software does not transmit Customer source code, secrets, or usage telemetry to Licensor, and contains no remote deactivation mechanism.
2. **Customer Data Ownership**: Customer retains all right, title, and interest in and to its source code, repositories, commit histories, security policies, bypass records, and audit logs. Licensor does not receive, access, process, or store Customer's source code or repository data during Software operation. Because Licensor does not receive or process Customer personal data or repository data, no third-party data sub-processors are engaged in Software execution.
3. **Offline Operation**: Once a valid License File is installed on the designated host, the Software operates for the duration of the licensed subscription term without requiring external network connectivity to Licensor's servers.
4. **Customer Infrastructure, Data Security & Confidentiality Responsibility**: Because all Software components and databases reside exclusively on Customer's self-hosted servers and Licensor has no access to Customer's environment, Customer is solely and exclusively responsible for securing its servers, network boundaries, administrative credentials, database backups, data confidentiality, and License Files. Licensor assumes no liability for any data loss, server compromise, or unauthorized disclosure resulting from Customer's infrastructure or operational security practices.

---

### Section 5. Mutual Confidentiality

1. **Confidentiality Obligations**: Each party ("Receiving Party") shall maintain the other party's ("Disclosing Party") Confidential Information in confidence, exercising at least the same degree of care it uses to protect its own confidential information of similar nature, and in no event less than reasonable care.
2. **Permitted Disclosure**: The Receiving Party shall use Confidential Information solely to exercise its rights and perform its obligations under this Agreement and shall not disclose Confidential Information to any third party, except to its employees, contractors, or professional legal and financial advisors who have a need to know and are bound by written confidentiality obligations at least as protective as those herein.
3. **Support Diagnostics**: Diagnostic bundles or log excerpts voluntarily submitted by Customer to Licensor for technical support shall be treated as Customer's Confidential Information, used solely to diagnose and resolve the reported issue, and deleted upon Customer's written request following ticket closure.
4. **Exceptions**: Confidentiality obligations shall not apply to information that: (a) is or becomes publicly available through no breach of this Agreement; (b) was lawfully known to the Receiving Party prior to disclosure without restriction; (c) is rightfully received from a third party without breach of any confidentiality obligation; or (d) is independently developed by the Receiving Party without use of or reference to the Disclosing Party's Confidential Information.

---

### Section 6. Intellectual Property & Infringement Indemnification

1. **Reservation of Rights**: Licensor retains all right, title, and interest, including all intellectual property rights, in and to the Software and Documentation. The Software is licensed, not sold, and no implied licenses are granted under this Agreement.
2. **Authority & Non-Infringement**: Licensor represents and warrants that (a) it is the lawful owner of the Software and has the legal authority to enter into this Agreement and grant the licenses set forth herein; (b) to Licensor's knowledge, the unmodified Software does not infringe any third-party copyright, patent, trademark, or proprietary right; and (c) it uses commercially reasonable efforts to ensure that official Software binaries, as delivered by Licensor, are free of malicious code or undocumented backdoors.
3. **IP Indemnification by Licensor**: Licensor shall defend Customer against any third-party claim, suit, or proceeding alleging that Customer's authorized use of the unmodified Software directly infringes a third-party copyright, patent, or proprietary right, and shall indemnify Customer for damages finally awarded by a court of competent jurisdiction or agreed to in a written settlement signed by Licensor. If such a claim occurs or is likely to occur, Licensor may, at its sole option and expense:
   - Procure for Customer the right to continue using the Software;
   - Modify or replace the Software so that it is non-infringing and functionally equivalent; or
   - If neither of the foregoing options is commercially reasonable, terminate the affected license and refund to Customer any prepaid, unused subscription fees on a pro-rata basis for the remainder of the active subscription term.
4. **Indemnification Exclusions**: Licensor shall have no obligation or liability under Section 6.3 for any claim arising from: (a) modification of the Software by anyone other than Licensor; (b) combination or use of the Software with third-party software, hardware, or data where the alleged infringement would not have occurred from the Software alone; or (c) use of the Software in breach of this Agreement. Sections 6.3 and 6.4 state Licensor's entire liability and Customer's sole and exclusive remedy for intellectual property infringement claims.

---

### Section 7. Usage Restrictions & License Safeguarding

1. **Restrictions**: Customer shall not, and shall not permit any third party to:
   - Sublicense, sell, resell, rent, lease, transfer, distribute, or provide the Software as a managed, hosted, or service-bureau offering to any third party;
   - Operate multiple concurrent production Master Node instances under a single License File, except during the fourteen (14) day migration grace period under Section 2.6;
   - Reverse engineer, decompile, disassemble, or attempt to derive the source code, underlying algorithms, or cryptographic keys of the Software, except to the minimum extent expressly permitted by applicable law notwithstanding this limitation;
   - Modify, patch, bypass, disable, or tamper with any license verification, cryptographic signature verification, seat quota enforcement, or host-binding mechanism in the Software;
   - Alter or forge digital signatures on License Files, policy envelopes, or exemption tokens; or
   - Remove, alter, or obscure any copyright, trademark, or proprietary notices in the Software or Documentation.
2. **Benchmarking**: Customer may conduct internal performance and accuracy benchmarks for its own evaluation. Customer shall not publish or disclose competitive benchmark results to third parties without Licensor's prior written consent so that configuration parity and testing methodology can be verified.
3. **License File Safeguarding**: Customer shall store its License File in strict confidence with file access restricted solely to authorized system administrators, and shall not commit, upload, publish, or expose its License File in any public repository, shared container image, or unauthorized environment.

---

### Section 8. Technical License Verification, Seat Quotas & Revocation

1. **Local Verification**: The Software verifies license validity, seat quotas, feature entitlements, and expiration dates locally using digital signatures and Node Fingerprint validation.
2. **System Clock Integrity**: Customer shall not manipulate or roll back host system clocks or virtualization timestamps to circumvent license expiration or migration grace period limits.
3. **Seat Quota Enforcement**: When active workstation enrollments reach the Authorized Seat quota, existing enrolled workstations continue operating and new workstation enrollments are rejected until existing seats are revoked or additional Authorized Seats are licensed.
4. **Revocation of Compromised or Leaked Licenses**: If any License File is discovered to be publicly exposed, leaked, shared outside Customer's organization, or used in breach of Section 7, Licensor reserves the right to permanently revoke and deactivate the affected License File across all Software releases and terminate the compromised license immediately without refund.

---

### Section 9. Operational Responsibility & Customer Indemnification

1. **Detection Limitations**: Automated credential detection relies on predefined signature patterns, regular expressions, and statistical heuristics. Licensor does not warrant or guarantee that the Software will detect all credentials, secrets, or security exposures, or that scan results will be free of false positives.
2. **Customer Security Governance**: Customer is solely responsible for:
   - Revoking and rotating any exposed credential with the applicable upstream service provider;
   - Reviewing and approving developer bypass requests and allowlist exclusions; and
   - Validating custom regular expressions authored by Customer prior to deployment.
3. **Repository History Modification**: While the Software creates safety backup references by default during repository history redaction, Customer is solely responsible for maintaining full external backups of target repositories and verifying modified commit histories before disabling safety backups, executing irreversible pruning operations, or force-pushing changes to remote repositories.
4. **Customer Indemnification**: Customer shall defend and indemnify Licensor against any third-party claim, suit, or proceeding arising out of: (a) Customer's breach of Section 7 (Usage Restrictions & License Safeguarding); or (b) Customer's source code, data, or content stored within Customer's repositories.

---

### Section 10. Maintenance, Updates & Technical Support

1. **Support Response Targets**: During an active subscription term, Licensor provides technical support via `crenoxhq@proton.me` during Business Hours (Monday through Friday, 09:00 to 18:00 UTC, excluding public holidays) with the following initial response targets based on Customer's subscribed tier:
   - **Enterprise Fleet Tier & Custom Enterprise Agreements**: Four (4) business hours for critical production incidents; twenty-four (24) business hours for standard inquiries;
   - **Business Tier**: Twenty-four (24) business hours;
   - **Team Tier**: Forty-eight (48) business hours;
   - **Evaluation Licenses**: Provided on an "AS IS" basis without response-time commitments.
2. **Updates**: Active commercial subscriptions include access to Software maintenance releases (across both Stable and Beta update channels), defect fixes, security patches, and signature catalog updates released during the subscription term.
3. **Scope of Support**: Licensor's support obligations are limited to defects in unmodified official Software binaries and Documentation. Licensor is not obligated to administer or troubleshoot Customer's operating systems, hardware, network infrastructure, third-party Git tooling, or Customer-authored regular expressions.

---

### Section 11. Limited Warranty & Disclaimers

1. **Limited Performance Warranty**: Licensor warrants that for a period of ninety (90) calendar days from initial delivery of the production License File ("Warranty Period"), the unmodified Software will perform substantially in accordance with the Documentation under normal operation. If Customer reports a reproducible material non-conformity in writing during the Warranty Period, Licensor shall, as Customer's sole and exclusive remedy, repair or replace the non-conforming Software, or if Licensor does not cure the non-conformity within thirty (30) calendar days of written notice, terminate the affected license and refund the fees paid for the affected subscription period.
2. **Disclaimer of Warranties**: EXCEPT FOR THE EXPRESS LIMITED WARRANTIES SET FORTH IN SECTIONS 6.2 AND 11.1, THE SOFTWARE AND DOCUMENTATION ARE PROVIDED "AS IS" AND LICENSOR DISCLAIMS ALL OTHER WARRANTIES, WHETHER EXPRESS, IMPLIED, STATUTORY, OR OTHERWISE, INCLUDING ANY IMPLIED WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE, TITLE, OR UNINTERRUPTED OR ERROR-FREE OPERATION.
3. **High-Risk Activities Exclusion**: The Software is not designed, certified, or licensed for use in hazardous or fail-safe environments requiring fail-safe performance, including nuclear facilities, aircraft navigation or communication systems, air traffic control, life-support machines, or weapons systems, where failure of the Software could lead directly to death, personal injury, or severe physical or environmental damage.

---

### Section 12. Limitation of Liability

1. **Exclusion of Consequential Damages**: TO THE MAXIMUM EXTENT PERMITTED BY APPLICABLE LAW, IN NO EVENT SHALL EITHER PARTY BE LIABLE TO THE OTHER FOR ANY INDIRECT, INCIDENTAL, SPECIAL, CONSEQUENTIAL, PUNITIVE, OR EXEMPLARY DAMAGES (INCLUDING LOSS OF PROFITS, LOSS OF REVENUE, BUSINESS INTERRUPTION, LOSS OF DATA, OR COST OF PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES) ARISING OUT OF OR RELATING TO THIS AGREEMENT, REGARDLESS OF THE THEORY OF LIABILITY (CONTRACT, TORT, STRICT LIABILITY, OR OTHERWISE), EVEN IF SUCH PARTY HAS BEEN ADVISED OF THE POSSIBILITY OF SUCH DAMAGES.
2. **Aggregate Liability Cap**: EXCEPT FOR BREACHES OF SECTION 5 (MUTUAL CONFIDENTIALITY), SECTION 7 (USAGE RESTRICTIONS & LICENSE SAFEGUARDING), OR CUSTOMER'S PAYMENT OBLIGATIONS UNDER SECTION 3, EACH PARTY'S TOTAL CUMULATIVE LIABILITY ARISING OUT OF OR RELATING TO THIS AGREEMENT SHALL NOT EXCEED THE TOTAL SUBSCRIPTION FEES ACTUALLY PAID BY CUSTOMER TO LICENSOR UNDER THIS AGREEMENT DURING THE TWELVE (12) MONTHS IMMEDIATELY PRECEDING THE EVENT GIVING RISE TO LIABILITY.
3. **Corporate Liability Boundary**: All obligations and liabilities under this Agreement attach solely to the contracting legal entities. No individual software engineer, developer, employee, officer, or director of either party shall bear personal liability under or in connection with this Agreement.

---

### Section 13. Seat Compliance Verification

1. **Verification Procedure**: Licensor shall not require remote or physical access to Customer's servers or source code repositories. Upon Licensor's written request, not more than once per twelve (12) month period, Customer shall provide a system status summary output within fifteen (15) business days verifying the number of active enrolled seats against the Authorized Seat quota.
2. **Excess Seat Resolution**: If the status summary demonstrates that active enrolled seats exceed the Authorized Seat quota, Customer shall within fifteen (15) business days either revoke excess workstation enrollments or execute an Order Schedule for the additional seats at Customer's contracted per-seat rate.

---

### Section 14. Term, Renewal & Termination

1. **Subscription Term**: This Agreement commences upon the earlier of Order Schedule execution or Software installation and continues for the subscription term specified in the License File, unless terminated earlier in accordance with this Section 14. Subscriptions renew upon mutual execution of a renewal Order Schedule and issuance of a renewal License File.
2. **Termination for Cause**: Either party may terminate this Agreement upon written notice if the other party commits a material breach of this Agreement and fails to cure such breach within thirty (30) calendar days after receiving written notice specifying the breach in reasonable detail; provided, however, that Licensor may terminate this Agreement immediately upon written notice in the event of Customer's breach of Section 7 or compromise/leakage of a License File under Section 8.4.
3. **Effect of Termination**: Upon expiration or termination of this Agreement for any reason:
   - All licenses granted hereunder shall immediately terminate, and Customer shall cease all use of the commercial Software features, uninstall Software hooks from its repositories, decommission the Master Node, and delete all copies of the License File in its possession or control;
   - Customer retains full ownership of all previously exported audit logs and its repository histories; and
   - Sections 1, 3 (with respect to accrued payment obligations), 4.2, 4.4, 5, 6.1, 6.4, 7, 8.4, 9.4, 11.2, 11.3, 12, 14.3, and 16 shall survive any expiration or termination of this Agreement.

---

### Section 15. Open Source Components

The compiled Software binaries include certain third-party open-source Go libraries licensed under permissive open-source licenses (including MIT, BSD, and Apache 2.0). Such open-source libraries are governed by their respective licenses, which do not restrict or alter the proprietary commercial licensing terms applicable to the Software as a whole.

---

### Section 16. General Provisions

1. **Entire Agreement & Precedence**: This Agreement, together with any mutually executed Order Schedule, constitutes the entire agreement between the parties concerning its subject matter and supersedes all prior or contemporaneous oral or written proposals, negotiations, or understandings. In the event of a conflict between this Agreement and a mutually signed Order Schedule, the Order Schedule shall control solely with respect to subscription tier, seat quantity, term dates, and fees. Any pre-printed or standard terms on a Customer purchase order or vendor portal are rejected and shall have no legal force or effect.
2. **Independent Contractors**: The parties are independent contractors. Nothing in this Agreement creates a partnership, joint venture, agency, fiduciary, or employment relationship between the parties.
3. **Assignment**: Neither party may assign or transfer this Agreement, in whole or in part, without the prior written consent of the other party, except that either party may assign this Agreement in its entirety upon written notice in connection with a merger, acquisition, corporate reorganization, or sale of all or substantially all of its assets, provided that any change in the Master Node host server remains subject to Section 2.6. Any attempted assignment in violation of this Section shall be void.
4. **Severability & Waiver**: If any provision of this Agreement is held by a tribunal of competent jurisdiction to be invalid, illegal, or unenforceable, such provision shall be modified to the minimum extent necessary to render it enforceable while preserving the parties' original intent, and the remaining provisions shall remain in full force and effect. No failure or delay by either party in exercising any right under this Agreement shall constitute a waiver of that right.
5. **Export Compliance**: Customer represents and warrants that it is not located in a jurisdiction subject to comprehensive trade embargoes and is not listed on any applicable government restricted or denied-party list, and shall not export, re-export, or use the Software in violation of applicable export control or sanctions laws.
6. **Force Majeure**: Neither party shall be liable for any delay or failure to perform its obligations under this Agreement (excluding payment obligations) to the extent caused by circumstances beyond its reasonable control, including acts of God, natural disasters, war, terrorism, civil unrest, embargoes, or widespread internet or telecommunications failures.
7. **Governing Law & Arbitration**: This Agreement shall be governed by and construed in accordance with the UNIDROIT Principles of International Commercial Contracts, without regard to conflict-of-law rules. Any dispute, controversy, or claim arising out of or relating to this Agreement, including its formation, validity, breach, or termination, that is not resolved through written negotiation between the parties within thirty (30) calendar days shall be finally settled by confidential binding arbitration under the Rules of Arbitration of the International Chamber of Commerce (ICC) by one (1) arbitrator appointed in accordance with said Rules. The language of the arbitration shall be English.

---

### Section 17. Notices & Contact Information

All legal notices, license re-issuance requests, or commercial inquiries under this Agreement shall be sent in writing to:
- **Licensor**: CrenoxHQ
- **Legal & Commercial Notices**: `crenoxhq@proton.me`
- **Official Repository**: `https://github.com/crenoxhq/crenox-pro`

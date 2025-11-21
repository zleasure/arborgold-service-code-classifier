# Service Code Classification Automation - Technical Deep Dive

## Context & Framing

**What This Document Is:**
I asked Claude to help me prepare for my Arborgold Data Specialist interview by creating a realistic, domain-specific data problem I might encounter during client migrations. Claude generated the problem scenario (inconsistent service codes across 127 variations), and I built the technical solution using approaches I've successfully implemented at Red West Analytics.

**Clear Attribution:**
- **Claude created**: The fictional migration scenario and the specific problem context
- **I created**: The technical solution architecture, Python implementation, and validation workflow based on my real production experience; relying heavily on pre-exsisted scripts, code and technical documentation from a previous real-world project

---

## The Problem (Claude-Generated Scenario)

During a typical Arborgold customer migration, I'd likely encounter **inconsistent service codes** across their legacy systems:

**The Scenario:**
A mid-size tree care company (Green Valley Tree Care) is migrating from Excel workbooks, QuickBooks, and ACT! CRM to Arborgold. Their Excel scheduling workbooks contain **127 unique service code variations**, but Arborgold's standard service catalog has only **18 documented service types**.

**Examples of unmapped codes:**
- `EMRG-STORM` → Emergency storm response?
- `Cbl-Brce` → Cable bracing?
- `PHC-SPRAY` → Plant health care spray?
- `CONSULT-ISA` → ISA arborist consultation?
- `TP`, `PRUNE`, `Tree_Prune` → All variations of "tree pruning"

**The Challenge:**
- Manual classification: 7-8 minutes per code × 127 codes = **~16 hours of work**
- Requires domain expertise we don't have (we're data specialists, not certified arborists)
- Risk of inconsistent classifications across similar codes
- Client can't go live until all codes are properly mapped

---

## My Real-World Experience (Red West Analytics)

**I faced a nearly identical challenge** at Red West Analytics with clinical trial data:

### The Actual Problem I Solved:

**Context:**
- Client was conducting clinical studies on microorganisms and pink eye treatments
- Lab data arrived with microorganism names and abbreviations
- Client had a specific taxonomy with threshold values for each organism
- Labs used different naming conventions and frequently sent new organisms not in our system

**The Pain Point:**
- Manual lookup and classification took hours
- Required biological expertise we didn't have
- New organisms appeared weekly
- Couldn't let unmapped organisms block the analysis pipeline

### My Solution Architecture:

1. **Built a crosswalk/dimension table** mapping lab names → client taxonomy
2. **Scraped authoritative microorganism databases** to create reference context
3. **Integrated OpenAI API** to classify unmapped values using that context
4. **Implemented human-in-the-loop validation** for quality control
5. **Created weekly client review process** to approve/correct AI suggestions

### Real Results:
- **90%+ reduction in manual effort** (from hours to minutes)
- **Zero misclassifications** after client validation
- **Self-improving system** - crosswalk grew smarter with each review cycle
- **Scalable pattern** - now handling 3 different clinical studies

**This exact approach transfers directly to Arborgold's service code problem.**

---

## The Technical Solution (My Implementation)

### Solution Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  STEP 1: Build Industry Reference Taxonomy                  │
│  → Scrape authoritative arboriculture websites              │
│  → Structure service types and categories                   │
│  → Create JSON reference for AI context                     │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 2: Create Crosswalk Table Infrastructure              │
│  → Design PostgreSQL table schema                           │
│  → Seed with known, manually verified mappings              │
│  → Include confidence scores and audit fields               │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 3: Identify Unmapped Service Codes                    │
│  → SQL query to find codes not in crosswalk                 │
│  → Sort by usage frequency (prioritize high-impact)         │
│  → Profile descriptions for additional context              │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 4: AI-Assisted Classification                         │
│  → Load taxonomy + existing mappings as context             │
│  → Build structured prompt with clear instructions          │
│  → Call OpenAI API with low temperature (consistency)       │
│  → Parse JSON response and calculate confidence scores      │
└────────────────────┬────────────────────────────────────────┘
                     ↓
┌─────────────────────────────────────────────────────────────┐
│  STEP 5: Human Validation & Feedback Loop                   │
│  → Export low/medium confidence items for review            │
│  → Client/stakeholder approves or corrects                  │
│  → Update crosswalk and mark as verified                    │
│  → Re-run transformations with approved mappings            │
└─────────────────────────────────────────────────────────────┘
```

---

## Implementation Details

### Why This Approach Works

**1. Balances Automation with Safety**
- AI handles the tedious research and pattern matching
- Human domain experts make final decisions on ambiguous cases
- Confidence scores help prioritize where to spend review time
- High-confidence classifications (≥90%) can often auto-approve

**2. Adapts to Client Variability**
- Every tree care company uses unique codes and abbreviations
- Can't hard-code all possible variations upfront
- System learns from each migration and gets smarter over time
- Crosswalk becomes a reusable asset across clients

**3. Production-Ready Design**
- Comprehensive error handling and logging
- Clear audit trail (who classified what, when, why)
- Rollback capability if classifications need correction
- Integrates seamlessly with existing ETL pipelines

**4. Scalable Pattern**
- Not specific to service codes - works for any categorical standardization
- Could adapt to: equipment types, material codes, customer segments, etc.
- Same pattern used at Red West for completely different domain (microbiology)

---

## Technical Decisions & Tradeoffs

### Why OpenAI API vs. Rules-Based Mapping?

**Rules-based approach:**
- Fast and deterministic
- No API costs
- Requires exhaustive manual mapping of all variations
- Brittle - breaks on unexpected formats
- Doesn't learn from context

**AI-assisted approach:**
- Handles novel variations without manual mapping
- Uses domain knowledge from training data
- Can reason about abbreviations and context
- Requires API key and incurs costs (~$0.50-2 per migration)
- Non-deterministic (same input might vary slightly)

**My decision:** AI-assisted with human validation gives best ROI for migrations.

### Why Low Temperature (0.1) in API Call?

Temperature controls randomness/creativity:
- **High temp (0.8-1.0)**: Creative, varied responses - good for content generation
- **Low temp (0.1-0.3)**: Consistent, conservative - good for classification

For service code classification, I want:
- Predictable, repeatable classifications
- Conservative confidence scores
- Minimal hallucination or creative interpretation

### Why Confidence Scores Matter

Not all classifications are equal. Confidence scoring allows:

**High Confidence (90-100%):**
- Clear, unambiguous matches
- Example: `EMRG-STORM` → "Emergency Storm Response"
- Can often auto-approve with spot-checking

**Medium Confidence (70-89%):**
- Likely correct but some ambiguity
- Example: `Cbl-Brce` → "Cabling and Bracing" (abbreviation guessing)
- Quick review recommended (5 minutes)

**Low Confidence (<70%):**
- Uncertain, needs domain expertise
- Example: `PHC-X12` → unclear what X12 refers to
- Requires careful manual review

This triaging saves significant time - focus human effort where it matters most.

---

## How Web Scraping Works (Technical Explanation)

**You might wonder:** "How can Python extract data from websites when there's no database, just text and pictures?"

### The Answer: HTML is Structured Data

1. **Every webpage is HTML** (HyperText Markup Language)
   - HTML uses tags like `<div>`, `<p>`, `<ul>`, `<li>` to structure content
   - Even though it *looks* like plain text, it's actually a structured tree

2. **Example HTML Structure:**
   ```html
   <div class="services">
     <h3>Tree Services</h3>
     <ul>
       <li>Tree Removal</li>
       <li>Tree Trimming</li>
       <li>Stump Grinding</li>
     </ul>
   </div>
   ```

3. **BeautifulSoup Parses This Structure:**
   ```python
   soup = BeautifulSoup(response.content, 'html.parser')
   
   # Find all list items under services section
   service_items = soup.find_all('li')
   
   # Extract just the text content
   services = [item.get_text().strip() for item in service_items]
   # Result: ['Tree Removal', 'Tree Trimming', 'Stump Grinding']
   ```

4. **We Navigate Like a File System:**
   - Find elements by tag name: `soup.find_all('li')`
   - Find by CSS class: `soup.find('div', class_='services')`
   - Navigate parent/child relationships: `div.find('ul').find_all('li')`

5. **Extract and Structure:**
   - Pull text content from HTML elements
   - Organize into dictionaries and lists
   - Save as JSON for the AI to consume

**For This Project Specifically:**
- We scrape professional tree care websites (Gabriel Tree Services, etc.)
- Extract their service listings to understand industry terminology
- Structure into categories the AI can reference
- This gives the AI "domain knowledge" it wouldn't otherwise have

---

## Code Walkthrough Highlights

### Key Python Functions

**1. Building the Taxonomy:**
```python
def build_isa_service_taxonomy() -> Dict[str, List[str]]:
    """
    Build service taxonomy based on ISA professional standards.
    
    ISA (International Society of Arboriculture) is the authoritative
    body for arboriculture certification. Their exam domains define
    the industry-standard service categories.
    """
    isa_services = {
        'Tree Services': [
            'Tree Removal', 'Tree Pruning', 'Stump Grinding', ...
        ],
        'Plant Health Care': [
            'Fertilization', 'Deep Root Injection', ...
        ],
        # ... more categories
    }
    return isa_services
```

**2. Calling OpenAI with Context:**
```python
def classify_with_openai(prompt: str) -> List[Dict]:
    """
    Call OpenAI API with structured prompt.
    
    Uses low temperature (0.1) for consistent classifications.
    Returns JSON array of classifications with confidence scores.
    """
    response = openai.ChatCompletion.create(
        model="gpt-4",
        messages=[
            {"role": "system", "content": "You are an arboriculture data specialist."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.1  # Conservative, consistent output
    )
    return json.loads(response.choices[0].message.content)
```

**3. Confidence-Based Reporting:**
```python
def generate_review_report(classifications: List[Dict]) -> Dict:
    """
    Segment classifications by confidence for prioritized review.
    
    High confidence: Auto-approve
    Medium confidence: Quick review
    Low confidence: Domain expert needed
    """
    high = [c for c in classifications if c['confidence'] >= 90]
    medium = [c for c in classifications if 70 <= c['confidence'] < 90]
    low = [c for c in classifications if c['confidence'] < 70]
    
    return {
        'high_confidence': {'count': len(high), 'recommendation': 'Auto-approve'},
        'medium_confidence': {'count': len(medium), 'recommendation': 'Quick review'},
        'low_confidence': {'count': len(low), 'recommendation': 'Expert review'},
    }
```

---

## Integration with Arborgold Migrations

### How This Fits into the Full ETL Pipeline

**Pre-Migration (This Tool):**
1. Extract raw service codes from client's Excel/CSV files
2. Run classification pipeline to build crosswalk table
3. Human validation and approval
4. Crosswalk ready for transformation step

**During Transformation:**
```sql
-- Use crosswalk to standardize service codes
SELECT 
    s.customer_name,
    s.service_code AS original_code,
    x.arborgold_service_name AS standardized_service,
    x.arborgold_category AS service_category
FROM staging.excel_schedules s
LEFT JOIN ref.service_code_crosswalk x 
    ON UPPER(TRIM(s.service_code)) = UPPER(x.source_code)
WHERE x.manually_verified = TRUE;
```

**Post-Migration:**
- All recurring schedules map to valid Arborgold services
- Reporting and analytics use consistent service categories
- Future migrations benefit from growing crosswalk library

---

## Lessons from Production Use

### What I Learned at Red West Analytics

**1. Always Keep Humans in the Loop**
- AI is powerful but not infallible
- Domain experts catch edge cases AI misses
- Weekly validation builds stakeholder trust

**2. Confidence Scores Are Critical**
- Not all classifications are created equal
- Scoring helps prioritize review effort efficiently
- High-confidence auto-approval saves massive time

**3. Document AI Reasoning**
- Store the AI's explanation in the database
- Helps humans understand why AI chose that classification
- Useful for debugging and improving prompts

**4. System Improves Over Time**
- First migration: 70% automation, lots of review
- Fifth migration: 90%+ automation, minimal review
- Crosswalk becomes valuable intellectual property

**5. Build for Auditability**
- Track who classified what, when, and why
- Enables rollback if classifications need correction
- Critical for regulated industries (pharma, healthcare)

---

## Future Enhancements

If I were extending this for production at Arborgold:

**1. Multi-Client Learning**
- Pool crosswalk mappings across all clients (anonymized)
- New clients benefit from historical patterns
- Build industry-wide service code dictionary

**2. Active Learning Loop**
- Track which AI classifications get corrected most often
- Retrain or improve prompts based on correction patterns
- Continuously improve confidence scoring accuracy

**3. Integration with Onboarding Dashboard**
- Visual UI for stakeholder review process
- Drag-and-drop interface for corrections
- Real-time progress tracking

**4. Synonym Detection**
- Automatically group similar codes before classification
- Reduces duplicate AI calls
- Example: `TP`, `T_PRUNE`, `TreePrune` → classify together

**5. Industry Taxonomy Auto-Updates**
- Scheduled scraping of ISA website for new services
- Alert when new industry standards emerge
- Keep taxonomy current without manual maintenance

---

## Conclusion

This classification automation demonstrates:
**Real production patterns** I've successfully deployed  
**Transfer of domain knowledge** from pharma to tree care  
**Thoughtful engineering decisions** balancing automation and safety  
**Scalable architecture** adaptable to other categorical standardization problems  
**Business value focus** - solving real pain points during migrations  

The approach isn't just theoretical - it's battle-tested in production and reduced manual effort by 90%+ while maintaining quality through human validation.

---

## Technical Q&A

**Q: Why not use fuzzy string matching instead of AI?**  
A: Fuzzy matching works for typos (`TreeRemoval` vs `Tree Removal`) but fails on abbreviations (`TR`, `EMRG-STORM`), synonyms, and contextual understanding. AI can reason about domain knowledge.

**Q: What if OpenAI API is down?**  
A: Fallback to manual classification queue. The pipeline is designed so AI is an accelerator, not a dependency. Humans can always classify manually.

**Q: How do you prevent AI hallucination?**  
A: (1) Low temperature for consistency, (2) Structured prompt with clear taxonomy, (3) Confidence scoring, (4) Human validation before production use.

**Q: Can this work for other industries?**  
A: Absolutely. The pattern is domain-agnostic. At Red West, I used it for microorganisms. At Arborgold, tree services. Could work for HVAC service codes, medical procedures, etc.

**Q: What's the cost per migration?**  
A: Roughly $0.50-2 in OpenAI API costs for 100-150 codes. Compare to 16 hours of manual work at $50/hr = $800. ROI is massive.

---

*Document created by Zack Leasure as a technical demonstration for the Arborgold Data Specialist role. The solution architecture is based on production experience at Red West Analytics.*

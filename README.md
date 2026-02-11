KEYWORD_EXTRACTION_PROMPT = """You are an expert at analyzing database query requests. Your task is to extract all potential keywords and entities from a user's natural language query, regardless of word order or format.

USER QUERY:
{user_query}

TASK:
Extract all potential keywords and classify them into categories. Users may phrase queries in any order, so you must identify entities based on meaning, not position.

IMPORTANT CONTEXT:
- The target column in the prompt is often a BUSINESS TERM (e.g., "FRS Business Unit") that will later map to a TECHNICAL COLUMN NAME (e.g., "FRS BU" or "FRS_BU")
- The table in the prompt is often a BUSINESS TERM (e.g., "Corporate Loans") that will later map to a TECHNICAL TABLE NAME (e.g., "STD_CLN_INB_DL")
- Extract the BUSINESS TERMS as they appear in the prompt - the fuzzy matching to technical names happens in the next step
- Focus on identifying what the user is referring to, even if it's phrased in business language

IMPORTANT CONTEXT:
- The target column name in the prompt may be a BUSINESS TERM (e.g., "FRS Business Unit") that maps to a TECHNICAL COLUMN NAME (e.g., "FRS BU")
- The table name in the prompt may be a BUSINESS TERM (e.g., "Corporate Loans") that maps to a TECHNICAL TABLE NAME (e.g., "STD_CLN_INB_DL")
- Extract the BUSINESS TERMS from the prompt - the mapping to technical names happens in the next step
- Focus on identifying what the user is referring to, even if it's phrased in business language

EXTRACTION RULES:
1. Extract ALL noun phrases and important terms that could refer to database entities
2. Consider business terminology, abbreviations, and technical names
3. Don't rely on word order - "column from table" and "from table column" mean the same thing
4. Extract terms that might be:
   - Column names (e.g., "FRS Business Unit", "customer_id", "amount")
   - Table names (e.g., "Corporate Loans", "customer_table", "orders")
   - Schema names (e.g., "Olympus Composition Layer", "sales_schema")
   - Domain names (e.g., "Customer Management", "Sales Domain")
   - Query actions (e.g., "produce error", "show", "find", "validate")
   - Conditions (e.g., "is null", "is blank", "equals", "greater than")

OUTPUT FORMAT (JSON):
{{
    "target_column": {{
        "text": "the column that the rule/validation applies to",
        "confidence": 0.0-1.0,
        "context_clues": ["why this is the target column"]
    }},
    "dataset_source": {{
        "schema": "schema name if specified (e.g., 'Olympus Risk managed Schema')",
        "table": "table name if specified",
        "domain": "domain name if specified",
        "confidence": 0.0-1.0
    }},
    "condition_columns": [
         {{
            "text": "column used in WHERE clause or filtering condition",
            "condition": "the condition (e.g., 'is null', 'equals X')",
            "confidence": 0.0-1.0
        }}
    ],
    "keywords": [
        {{
            "text": "exact phrase from query",
            "type": "table|schema|domain|action|condition_phrase|ambiguous",
            "confidence": 0.0-1.0,
            "context_clues": ["reasoning for classification"]
        }}
    ],
    "query_intent": "brief description of what user wants to do",
    "extracted_phrases": [
        "all noun phrases and important terms"
    ]
}}

CLASSIFICATION GUIDELINES:

**CRITICAL DISTINCTION - TARGET COLUMN vs CONDITION COLUMNS:**
- **TARGET COLUMN**: The PRIMARY column that the rule/validation needs to be applied ON. This is the column being validated/checked.
  - Examples: "FRS Business Unit" in "Produce Error if FRS Business Unit is null"
  - Indicators: Column mentioned with validation action ("is null", "is blank", "produce error for [X]")
  - This is the MOST IMPORTANT column to identify - it's what the rule is about!
  
- **CONDITION COLUMNS**: Columns used in WHERE clauses for filtering/conditions (NOT the target of validation)
  - Examples: "where status = 'active'" → "status" is a condition column, NOT the target
  - Indicators: Appears after "where", "and", "or" in filtering contexts
  - These are used to FILTER the dataset, not to validate

**DATASET SOURCE:**
- **Schema indicators**: "use dataset from [X] schema", "from [X] schema", "in [X] Layer", terms with "Schema" or "Layer"
- **Table indicators**: "from [table]", "in [table]", "use [table]"
- **Domain indicators**: High-level business domains, organizational units

**OTHER ENTITIES:**
- **Action indicators**: Verbs like "produce", "show", "find", "validate", "check", "generate"
- **Condition phrases**: "is null", "is blank", "equals", "not equal", comparison operators (these describe HOW to validate, not which column)
- **Ambiguous**: When unsure, mark as ambiguous - we'll search in multiple entity types

**PRIORITY:**
1. FIRST identify the TARGET COLUMN (the one the rule applies to)
2. THEN identify dataset source (schema/table/domain)
3. THEN identify condition columns (if any, used in WHERE clauses)
4. Extract other keywords for context


Example 1:
Input: "Produce error When Financial Reporting System (FRS) Affiliate Code for om_fin_rwa_aggregator_fact is populated and values in not equal to '00000', the customer must be an active third party global finance customer Identifier in Account Master Central (AMC) Reference Table"

Output:
{
    "main_column": {
        "schema": null,
        "table": "om_fin_rwa_aggregator_fact",
        "column": "Financial Reporting System (FRS) Affiliate Code"
    },
    "supporting_columns": [
        {
            "schema": null,
            "table": "Account Master Central (AMC) Reference",
            "column": "active third party global finance customer Identifier"
        }
    ]
}

Example 2:
Input: "Validate Customer Status Code from olympus consumption for customer_master where Region Code equals 'US'"

Output:
{
    "main_column": {
        "schema": "olympus consumption",
        "table": "customer_master",
        "column": "Customer Status Code"
    },
    "supporting_columns": [
        {
            "schema": null,
            "table": null,
            "column": "Region Code"
        }
    ]
}

Example 3:
Input: "Check Product Identifier for inventory_table where Quantity on Hand is greater than Minimum Stock Level"

Output:
{
    "main_column": {
        "schema": null,
        "table": "inventory_table",
        "column": "Product Identifier"
    },
    "supporting_columns": [
        {
            "schema": null,
            "table": null,
            "column": "Quantity on Hand"
        },
        {
            "schema": null,
            "table": null,
            "column": "Minimum Stock Level"
        }
    ]
}

Now extract from the user's query following the same pattern."""

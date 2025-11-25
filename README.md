import sqlglot
import re
from typing import Dict, List

def identify_partition_keys(sql: str) -> List[Dict]:
    """
    Identify likely partition keys from SQL query.
    Checks both CTE WHERE clauses and main query WHERE.
    """
    tree = sqlglot.parse_one(sql)
    
    column_scores = {}
    
    # Check CTEs first (WITH clauses) - partition filters often here
    ctes = tree.find_all(sqlglot.expressions.CTE)
    for cte in ctes:
        cte_query = cte.this
        where_clause = cte_query.find(sqlglot.expressions.Where)
        if where_clause:
            analyze_where_clause(where_clause, column_scores, is_cte=True)
    
    # Check main query WHERE
    main_where = tree.find(sqlglot.expressions.Where)
    if main_where:
        analyze_where_clause(main_where, column_scores, is_cte=False)
    
    # Convert to sorted list
    results = []
    for col_name, info in column_scores.items():
        results.append({
            'column': col_name,
            'score': info['score'],
            'filter_type': info.get('filter_type'),
            'value': info.get('value', ''),
            'likely_partition_key': info['score'] >= 8,
            'possibly_partition_key': 5 <= info['score'] < 8
        })
    
    results.sort(key=lambda x: x['score'], reverse=True)
    return results


def analyze_where_clause(where_clause, column_scores, is_cte=False):
    """
    Analyze a WHERE clause and update column_scores.
    """
    condition_order = 0
    
    # Get all conditions (use find_all to recursively find conditions)
    # This will find In, EQ, etc. throughout the WHERE tree
    for node in where_clause.walk():
        condition_order += 1
        
        if isinstance(node, sqlglot.expressions.In):
            # Pattern: col IN (val1, val2, ...)
            column = node.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                values = node.expressions
                
                score = 5  # IN clause bonus
                score += analyze_column_name(col_name)
                score += analyze_values(values)
                if condition_order == 1:
                    score += 1
                if is_cte:
                    score += 1  # CTE bonus (partition filters often in CTEs)
                
                update_score(column_scores, col_name, score, 'IN', f"{len(values)} values")
        
        elif isinstance(node, sqlglot.expressions.EQ):
            # Pattern: col = value
            left = node.this
            right = node.expression
            
            if isinstance(left, sqlglot.expressions.Column):
                col_name = left.sql()
                value = right
                
                score = 3  # Equality bonus
                score += analyze_column_name(col_name)
                score += analyze_single_value(value)
                if condition_order == 1:
                    score += 1
                if is_cte:
                    score += 1
                
                value_str = value.sql() if value else ''
                update_score(column_scores, col_name, score, 'EQUALS', value_str)
        
        elif isinstance(node, sqlglot.expressions.Between):
            # Pattern: col BETWEEN val1 AND val2
            column = node.this
            if isinstance(column, sqlglot.expressions.Column):
                col_name = column.sql()
                
                score = 2  # BETWEEN bonus
                score += analyze_column_name(col_name)
                if condition_order == 1:
                    score += 1
                if is_cte:
                    score += 1
                
                update_score(column_scores, col_name, score, 'BETWEEN', '')


def update_score(column_scores, col_name, score, filter_type, value):
    """Update column score if higher than existing."""
    if col_name not in column_scores or score > column_scores[col_name]['score']:
        column_scores[col_name] = {
            'score': score,
            'filter_type': filter_type,
            'value': value
        }


def analyze_column_name(col_name: str) -> int:
    """Score based on column name patterns."""
    col_upper = col_name.upper()
    score = 0
    
    # Date/time indicators
    if any(keyword in col_upper for keyword in ['DATE', 'DT', 'TIME', 'YEAR', 'MONTH', 'DAY']):
        score += 3
    
    # Categorical partitioning indicators
    elif any(keyword in col_upper for keyword in ['REGION', 'COUNTRY', 'TYPE', 'CATEGORY', 'STATUS']):
        score += 1
    
    return score


def analyze_values(values: List) -> int:
    """Score based on value patterns (for IN clause)."""
    if not values or len(values) < 3:
        return 0
    
    score = 0
    sample_values = [v.sql() for v in values[:5]]
    
    # 8-digit integers (YYYYMMDD)
    if all(re.match(r'^\d{8}$', str(v).strip("'\"")) for v in sample_values):
        score += 2
    # Date strings
    elif all(re.match(r'^\d{4}-\d{2}-\d{2}', str(v).strip("'\"")) for v in sample_values):
        score += 2
    # Short categorical
    elif all(len(str(v).strip("'\"")) <= 10 for v in sample_values):
        score += 1
    
    return score


def analyze_single_value(value) -> int:
    """Score based on single value pattern."""
    if not value:
        return 0
    
    val_str = value.sql().strip("'\"")
    score = 0
    
    # 8-digit integer (YYYYMMDD)
    if re.match(r'^\d{8}$', val_str):
        score += 2
    # Date string
    elif re.match(r'^\d{4}-\d{2}-\d{2}', val_str):
        score += 2
    # Short categorical
    elif len(val_str) <= 10:
        score += 1
    
    return score


# Test with your query
sql = """
with CACHE_OM_DEBT_COMMON_BALANCE as ( select common_balance_sk as flow_entity_sk ,uipid as flow_entity_id ,source_system_id as src_system ,common_balance_sk ,uipid ,source_system_id ,transaction_currency_code , transactional_amount, DWH_BUSINESS_DATE, SOURCE_AS_OF_DATE, base_amount, FDL_ACCOUNT_ID, frs_position_unit_code, frs_affiliate_code, firm_account_sk, BALANCE_TYPE FROM gfolyr2b_MANAGED.om_debt_common_balance where DWH_BUSINESS_DATE = 20251121 ) , CACHE_OM_BALANCE_TYPE_DIM_ACTV as ( SELECT balance_type_code FROM gfolynref_standardization.OM_BALANCE_TYPE_DIM_ACTV ) select src_system, flow_entity_id, flow_entity_sk from CACHE_OM_DEBT_COMMON_BALANCE T LEFT JOIN CACHE_OM_BALANCE_TYPE_DIM_ACTV REF1 ON T.BALANCE_TYPE = REF1.BALANCE_TYPE_CODE Where T.BALANCE_TYPE is NOT null AND REF1.BALANCE_TYPE_CODE IS NULL
"""

results = identify_partition_keys(sql)

print("Partition Key Analysis:")
print("="*70)
for result in results:
    print(f"\nColumn: {result['column']}")
    print(f"  Score: {result['score']}")
    print(f"  Filter Type: {result['filter_type']}")
    print(f"  Value: {result['value']}")
    if result['likely_partition_key']:
        print(f"  ✅ Very likely partition key")
    elif result['possibly_partition_key']:
        print(f"  ⚠️  Possibly partition key")
    else:
        print(f"  ❌ Unlikely partition key")

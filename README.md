import sqlglot
import re

sql = """
with CACHE_OM_DEBT_COMMON_BALANCE as ( select common_balance_sk as flow_entity_sk ,uipid as flow_entity_id ,source_system_id as src_system ,common_balance_sk ,uipid ,source_system_id ,transaction_currency_code , transactional_amount, DWH_BUSINESS_DATE, SOURCE_AS_OF_DATE, base_amount, FDL_ACCOUNT_ID, frs_position_unit_code, frs_affiliate_code, firm_account_sk, BALANCE_TYPE FROM gfolyr2b_MANAGED.om_debt_common_balance where DWH_BUSINESS_DATE = 20251121 ) , CACHE_OM_BALANCE_TYPE_DIM_ACTV as ( SELECT balance_type_code FROM gfolynref_standardization.OM_BALANCE_TYPE_DIM_ACTV ) select src_system, flow_entity_id, flow_entity_sk from CACHE_OM_DEBT_COMMON_BALANCE T LEFT JOIN CACHE_OM_BALANCE_TYPE_DIM_ACTV REF1 ON T.BALANCE_TYPE = REF1.BALANCE_TYPE_CODE Where T.BALANCE_TYPE is NOT null AND REF1.BALANCE_TYPE_CODE IS NULL
"""

def identify_partition_keys(sql: str):
    tree = sqlglot.parse_one(sql)
    column_scores = {}
    
    # Check CTEs
    ctes = tree.find_all(sqlglot.expressions.CTE)
    for cte in ctes:
        cte_query = cte.this
        where_clause = cte_query.find(sqlglot.expressions.Where)
        if where_clause:
            for node in where_clause.walk():
                if isinstance(node, sqlglot.expressions.EQ):
                    left = node.this
                    right = node.expression
                    if isinstance(left, sqlglot.expressions.Column):
                        col_name = left.sql()
                        value = right.sql() if right else ''
                        
                        score = 3  # EQ
                        col_upper = col_name.upper()
                        if any(k in col_upper for k in ['DATE', 'DT', 'TIME']):
                            score += 3
                        val_str = value.strip("'\"")
                        if re.match(r'^\d{8}$', val_str):
                            score += 2
                        score += 1  # CTE bonus
                        
                        if col_name not in column_scores or score > column_scores[col_name]['score']:
                            column_scores[col_name] = {
                                'score': score,
                                'filter_type': 'EQUALS',
                                'value': value
                            }
    
    # Check main WHERE
    main_where = tree.find(sqlglot.expressions.Where)
    if main_where:
        for node in main_where.walk():
            if isinstance(node, sqlglot.expressions.EQ):
                left = node.this
                if isinstance(left, sqlglot.expressions.Column):
                    col_name = left.sql()
                    if col_name not in column_scores:
                        column_scores[col_name] = {
                            'score': 0,
                            'filter_type': 'EQUALS',
                            'value': ''
                        }
    
    results = []
    for col_name, info in column_scores.items():
        results.append({
            'column': col_name,
            'score': info['score'],
            'filter_type': info['filter_type'],
            'value': info['value'],
            'likely_partition_key': info['score'] >= 8
        })
    
    results.sort(key=lambda x: x['score'], reverse=True)
    return results

results = identify_partition_keys(sql)

print("Partition Key Analysis:")
print("="*70)
for result in results:
    print(f"\nColumn: {result['column']}")
    print(f"  Score: {result['score']}")
    print(f"  Filter: {result['filter_type']}")
    print(f"  Value: {result['value']}")
    if result['likely_partition_key']:
        print(f"  ✅ LIKELY PARTITION KEY")
    else:
        print(f"  ❌ Unlikely partition key")

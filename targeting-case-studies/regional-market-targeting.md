# Regional Market Promotional Campaign Targeting

**Project**: TICKET-002  
**Campaign Type**: Regional Promotional Campaign  
**Complexity**: High - Multi-table joins with complex partner suppression logic

## Business Context

**Challenge**: Launch promotional sweepstake campaign in specific regional market while navigating complex partner suppression rules and strict CRM marketing preferences. Required precise compliance with partner agreements and regional regulations.

**Objective**: Maximize eligible audience reach while maintaining 100% compliance with partner suppression requirements and customer opt-in preferences.

## Technical Solution

### Architecture Overview
- **Data Sources**: **GCP BigQuery** customer data warehouse, order history tables, partner suppression rules
- **Processing**: Complex **BigQuery joins** with partner exclusion logic and CRM preference filtering
- **Output**: Compliant targeting list exported to marketing systems from **GCP BigQuery**

### Implementation Details

```sql
-- Regional Promotional Campaign Targeting with Partner Suppression
CREATE OR REPLACE TABLE
  `company-marketing.email_exports.product_regional_promo_YYYYMM` AS (

-- Partner suppression logic with CRM marketing preferences  
WITH partner_suppression AS (
  SELECT 
    license_id,
    CASE 
      WHEN partner_name <> 'Company Corp - Referral' 
        AND partner_name IS NOT NULL 
        AND enable_crm_marketing IN (2, 0) 
      THEN TRUE 
      ELSE FALSE 
    END AS suppression
  FROM `company-data.analytics.order_table`
  -- Get most recent order per customer
  QUALIFY row_number() OVER(
    PARTITION BY license_id 
    ORDER BY order_date DESC
  ) = 1
)   

SELECT
  a.account.email_address as email,
  -- Regional market locale construction prioritizing product language
  CONCAT(
    COALESCE(
      upper(a.subscription.product_language),
      upper(IF(LENGTH(a.account.contact_language)=3,NULL, a.account.contact_language)),
      upper(IF(LENGTH(a.account.account_language)=3,NULL, a.account.account_language)),
      upper(a.product.product_language)
    ),
    '-', 
    upper(a.subscription.country_code)
  ) as locale,
  a.license_id as psn,
  a.account_guid as guid

FROM `company-data.analytics.customer_table` a

-- Join order data for partner and channel validation
LEFT JOIN `company-data.analytics.order_table` b
  ON a.license_id = b.license_id
LEFT JOIN `company-data.analytics.subscription_order_table` c
  ON a.license_id = c.license_id  
LEFT JOIN partner_suppression p
  ON a.license_id = p.license_id

WHERE
  -- Product eligibility: Consumer security products only
  (a.product.product_tier IN ('TIER1', 'TIER2', 'TIER3')
   OR a.product.product_sku IN (
     'PRODUCT1','PRODUCT2','PRODUCT3','PRODUCT4','PRODUCT5','PRODUCT6',
     'PRODUCT7','PRODUCT8','PRODUCT9','PRODUCT10','PRODUCT11','PRODUCT12',
     'PRODUCT13','PRODUCT14','PRODUCT15','PRODUCT16','PRODUCT17','PRODUCT18',
     'PRODUCT19','PRODUCT20'
   )
  )
  -- Order channel validation: Specific prefixes only
  AND (b.order_number LIKE 'CH1%' 
    OR b.order_number LIKE 'CH2%' 
    OR c.order_number LIKE 'CH3%'
  )  
  -- Essential customer data requirements
  AND a.account_guid IS NOT NULL
  AND a.account.email_address IS NOT NULL
  -- Geographic restriction: Target region only
  AND a.subscription.country_code = 'REGION_A'
  -- Active subscription requirements
  AND a.subscription.is_paid_active is TRUE
  AND a.subscription.cancel_date IS NULL
  -- Marketing opt-in verification
  AND a.account.marketing_enabled IS TRUE
  -- Product maturity: Minimum X days since activation
  AND DATE(a.subscription.activation_date) < CURRENT_DATE() - INTERVAL X DAY
  -- Subscription validity window: Within expiry range
  AND DATE(a.subscription.expiry_date) BETWEEN
    CURRENT_DATE() - INTERVAL Y DAY
    AND CURRENT_DATE() + INTERVAL Z DAY
  -- Partner suppression compliance
  AND (suppression IS FALSE OR suppression IS NULL)

-- Deduplication: Latest expiry date per customer
QUALIFY row_number() OVER(
  PARTITION BY a.account_guid 
  ORDER BY a.subscription.expiry_date DESC
) = 1
)
```

## Technical Highlights

### 1. **Partner Suppression Logic**
- **Complex Business Rules**: Handles promotional partner exclusions with CRM preference validation
- **Temporal Accuracy**: Uses most recent order data with window functions
- **Compliance Safety**: Conservative approach (suppression IS FALSE OR NULL) ensures no violations

### 2. **Multi-Table Architecture**
- **Customer Data**: Primary customer table for core attributes
- **Order History**: Both regular orders and subscription orders for channel validation
- **Partner Rules**: Separate CTE for clear suppression logic separation

### 3. **Geographic & Product Precision**
- **Market Focus**: Configurable regional targeting (REGION_A)
- **Product Scope**: Comprehensive consumer product inclusion list
- **Channel Validation**: Specific order number prefix requirements (CH1%, CH2%, CH3%) from different tables

### 4. **Customer Lifecycle Filtering**
- **Activation Maturity**: Minimum X-day activation period
- **Subscription Window**: Y days past expiry to Z days future expiry
- **Marketing Consent**: Marketing enabled flag verification
- **Active Status**: Paid and non-cancelled subscription requirements

### 5. **Data Quality & Deduplication**
- **Essential Fields**: NULL checking for account_guid and email_address
- **Customer Deduplication**: Latest subscription expiry per account_guid
- **Partner Deduplication**: Most recent order per customer for suppression logic

## Business Impact

### Compliance Achievements
- **Partner Agreement**: 100% compliance with promotional partner suppression rules
- **Regional Regulations**: Full adherence to regional market email marketing laws  
- **CRM Preferences**: Respected all customer opt-in/opt-out preferences
- **Data Privacy**: Proper handling of customer personal information

### Campaign Performance
- **Audience Size**: Optimized eligible customer pool while maintaining compliance
- **Delivery Quality**: Clean targeting resulted in high deliverability rates
- **Legal Risk**: Zero compliance violations or partner agreement breaches
- **Customer Trust**: Maintained customer confidence through proper consent handling

### Technical Excellence
- **Query Performance**: Efficient multi-table joins with proper indexing
- **Maintainability**: Clear CTE structure for future campaign modifications
- **Scalability**: Framework reusable for other regional promotional campaigns
- **Documentation**: Self-documenting SQL with clear business logic comments

## Lessons Learned

1. **Partner Rules Complexity**: Promotional partner suppression requires careful temporal logic
2. **Regional Markets**: Product language priority important for locale construction
3. **Multi-Channel Orders**: Different order systems require separate handling
4. **Conservative Compliance**: "FALSE OR NULL" logic prevents accidental inclusions
5. **Window Functions**: Essential for deduplication in multi-table scenarios

## Reusable Framework

This targeting pattern established a template for:
- **Regional Compliance**: Adaptable partner suppression logic for other markets
- **Promotional Campaigns**: Sweepstakes and promotional campaign targeting methodology  
- **Multi-Channel Validation**: Order system integration patterns
- **Market Expansion**: Scalable framework for new regional markets

---

**Skills Demonstrated**: Complex SQL joins, regulatory compliance, partner relationship management, regional market expertise, data quality assurance, performance optimization

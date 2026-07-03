---
name: amazon-return-review-analysis
description: Analyze Amazon return reports and buyer review data by product category, grouping rows with the first 8 characters of SKU inside each category, summarizing return reasons, optionally calculating targeted return rates, using reviews only to validate return-reason hypotheses, and producing a final Excel workbook (.xlsx). Use when the user asks to analyze Amazon return reports, buyer return comments, reviews, return rates, SKU-level return optimization, Excel/XLSX return reports, or category-specific operations/product recommendations.
---

# Amazon Return And Review Analysis

## Purpose

Analyze Amazon return reports at the monthly product level. Group rows by product category first, then by the first 8 characters of SKU inside each category. Summarize buyer return notes into category-specific return-reason themes, and use buyer review data only as supporting evidence to validate or challenge those return conclusions. Treat sales/order data as optional and targeted: use it only when available or when the user needs true return-rate ranking for selected SKUs/months. Produce a final Excel workbook where each product category is its own worksheet/mini-report with overview, grouped SKU performance, return-reason detail, review-assisted judgment, and separate Amazon operations/product action items.

## Inputs

Accept spreadsheets, CSV files, copied tables, or pasted rows.

Required:
- Return report with SKU and return quantity or return records. Prefer reports that include both return date and order date.

Optional but preferred:
- Buyer review report with SKU or ASIN plus review rating, review title/content, review date, and marketplace when available.
- Sales/order report with SKU and sales volume, order volume, shipped units, or ordered units. Do not require a full historical order table by default; prefer SKU-month summary data for only the months and SKU groups needed to calculate targeted return rates.

Infer likely columns from names such as:
- Return report: `sku`, `seller sku`, `merchant sku`, `return date`, `return time`, `order date`, `purchase date`, `return quantity`, `quantity`, `buyer comment`, `buyer note`, `customer comments`, `return reason`, `disposition`, `asin`, `order id`, `marketplace`, `退货时间`, `订购时间`.
- Sales/order report: `sku`, `msku`, `seller sku`, `asin`, `date`, `month`, `order month`, `ordered units`, `units ordered`, `sales volume`, `sessions`, `orders`, `order count`, `销量`, `订单量`, `销售件数`, `销售数量`, `月份`.
- Review report: `sku`, `asin`, `rating`, `star`, `review title`, `review content`, `review text`, `body`, `date`, `buyer review`, `评论内容`, `星级`.

Ask a brief clarification only when the SKU column cannot be identified, or when multiple candidate sales/order quantity columns would materially change return-rate calculations.

## Matching Rules

- Create `group_sku` from the first 8 characters of SKU after trimming whitespace in every input table that has SKU.
- For return-rate calculations, prefer order-month attribution: use the return report's order date/purchase date to create `order_month`, then match returns to the sales/order table for that same `group_sku + order_month`.
- Use return date/return month to measure return-processing pressure, not as the default denominator month for return rate.
- If order date is missing in the return report, fall back to return month only after stating that the return rate may be distorted by Amazon's return-window lag.
- If a review table has ASIN but no SKU, match reviews by ASIN only when the return or sales table contains ASIN. State that ASIN-based matching is weaker than SKU-based matching if one ASIN maps to multiple `group_sku` values.
- Use review data only when it matches a `group_sku` or ASIN that appears in the return report. Do not add review-only SKU rows or review-only product categories to the main analysis; list unmatched reviews only as a data-coverage note if useful.
- Aggregate all matched data to `group_sku`; when month fields exist, also aggregate to `group_sku + order_month` before comparing return rates.
- Use product category as the primary report unit. After the shared analysis-scope section, create one top-level section per product category and repeat the same internal framework for that category. Do not organize sections 2-5 globally across all categories.
- Preserve child SKU, ASIN, marketplace, and date details when available so anomalies can be traced back.
- Do not mix different order months unless the user explicitly requests it. If return, sales, and review files cover different periods, state the date ranges and treat comparisons as directional.

## Return Window Rules

- Amazon returns commonly lag the sale because buyers have a return window. Do not default to `return_month returns / same return_month sales`.
- Treat `return_month` as the month Amazon processed or recorded the return.
- Treat `order_month` as the month that should receive the return-rate numerator when order date exists.
- Before calculating return rate, inspect the return report's order-date range. If the range is very broad or reaches far into the past, do not request a full historical sales/order export unless the user explicitly wants exact historical rates.
- For the default report, sales/order data is optional. If not provided, complete the return-reason analysis and label true return rate as unavailable.
- For targeted return-rate validation, request SKU-month sales/order summaries only for the prioritized `group_sku` values and relevant `order_month` values, not the entire order history.
- For a monthly return report, a practical default is the return month plus the previous 1-2 months of sales/order data. If older order months appear, list them as historical/exception returns unless matching sales summaries are already available.
- If sales/order data does not cover all order months found in the return report, calculate rates only for matched months and list unmatched order months as a data gap, not a blocker.
- Flag returns whose order date is much earlier than the normal return window as `历史/异常归因`; exclude them from default return-rate calculations unless the user supplies matching historical sales data.
- Mark order-month return rates by maturity:
  - `已成熟`: the order month ended at least 60 days before the analysis date or latest data date.
  - `部分成熟`: the order month ended 30-59 days before the analysis date or latest data date.
  - `未成熟`: the order month ended less than 30 days before the analysis date or latest data date.
- Prefer mature or partially mature order months for product severity conclusions. For immature months, report early signals but avoid firm conclusions.

## Workflow

1. Inspect every input table and identify the role of each file: return report, sales/order report, review report, or unknown support table.
2. Normalize each table:
   - Create `group_sku` from SKU first 8 characters.
   - Create `return_month` from return date when present.
   - Create `order_month` from order date/purchase date when present.
   - Convert return quantity, ordered units, sales units, order count, and rating fields to numeric values.
   - Treat empty buyer return comments as `未填写备注`.
   - Preserve original platform return reason separately from buyer free-text remarks.
3. Aggregate the return report by `group_sku`:
   - Return quantity or return record count.
   - Unique child SKUs and ASINs.
   - Top platform return reasons.
   - Buyer return remark themes.
   - Return-processing volume by `return_month`.
   - Return-rate numerator by `order_month` when order date exists.
   - Product category from the return report. Use this as the top-level reporting dimension.
4. If sales/order data exists, aggregate by `group_sku + sales/order month` when month data exists, and calculate:
   - `return_rate_by_units = order_month_return_quantity / same_order_month_sales_units` when sales or shipped units are available.
   - `return_rate_by_orders = order_month_return_records / same_order_month_order_count` when order count is available.
   - Use the unit-based return rate as primary when both unit and order denominators exist.
   - Mark missing or zero denominators as `无法计算`.
   - If only monthly sales files are provided, match each return to the sales month corresponding to the return's `order_month`, not the `return_month`.
   - If the needed sales/order data would require a large historical export, skip rate calculation by default and recommend a targeted SKU-month pull for only the highest-priority groups.
5. If review data exists, aggregate by `group_sku` and calculate:
   - Review count, average rating, low-rating count, and low-rating share. Treat 1-3 star reviews as low-rating unless the user gives another standard.
   - Review themes from title/content, using the same theme taxonomy as return comments when possible.
   - Evidence alignment: whether review complaints support, partially support, or conflict with return-reason themes.
   - Use review evidence inside the matching category/SKU analysis only. Do not create a standalone review-analysis section unless the user explicitly asks for it.
   - Exclude review-only `group_sku` values from category and SKU tables because they do not explain observed returns in the provided return report.
6. Classify buyer return remarks and review themes using the report language. Prefer specific labels over generic labels:
   - 尺寸/适配问题
   - 质量/做工问题
   - 功能不符合预期
   - 与图片或描述不符
   - 物流/包装损坏
   - 误购/不需要
   - 配件缺失
   - 使用体验问题
   - 无有效备注
   - 其他
7. Prioritize `group_sku` values using both concentration and risk:
   - High return quantity plus high return rate is highest priority.
   - High return quantity but average return rate may indicate sales-volume effect; still optimize, but avoid overstating product severity.
   - Low return quantity but high return rate may indicate a niche or new-SKU risk; mark as sample-size sensitive.
   - Return themes supported by low-star reviews have stronger evidence than return reasons alone.
8. Produce action lists inside each product-category section. Keep action items separated by product category, not as one mixed global table. For each action item, split recommendations into two tracks:
   - 亚马逊运营维度: specific listing copy, images/video, variation structure, size chart, QA content, review mining, packaging claims, buyer expectation management, return-policy monitoring, ads/search-term alignment.
   - 产品维度: specific product design, sizing/fit, material, durability, assembly/use flow, accessories, packaging protection, QC checks, supplier feedback, next production batch changes.
   - Each action item must include a concrete next step or acceptance check such as the exact image/video to add, the claim to revise, the QC test to run, the sample size to inspect, or the metric to review next month.
9. Export the final analysis as an Excel workbook (`.xlsx`) as the primary deliverable. Use Markdown, JSON, or CSV only as internal/source drafts when useful; do not stop at Markdown unless the user explicitly asks for Markdown only.

## Excel Output Rules

- Final output must include an Excel workbook path ending in `.xlsx`. Present the Excel workbook as the primary deliverable. Mention Markdown or other source files only as secondary artifacts when useful or explicitly requested.
- Use a workbook structure that is easy for an operator to filter, copy, and assign:
  - `目录`: list each report section, product category, priority level when available, and linked/visible worksheet name.
  - `分析口径`: data ranges, SKU grouping rule, matching rules, return-window treatment, review usage rule, and data limitations.
  - One worksheet per product category, named like `2-舞蹈包P0` or another short readable name within Excel's sheet-name limits.
  - Inside each product-category worksheet, keep the same mini-report framework: `分类概览`, `归类SKU表现`, `退货原因与SKU明细`, `Review辅助判断`, and `运营端与产品端行动清单`.
- Use real spreadsheet tables or clearly separated tabular ranges for `归类SKU表现` and `运营端与产品端行动清单`.
- Keep long bilingual buyer comments and action items readable:
  - Enable text wrapping.
  - Use practical column widths and row heights.
  - Freeze useful header rows.
  - Avoid clipped columns, hidden key fields, or action items squeezed into unreadable cells.
  - Split very wide or long sections into adjacent ranges only when it improves readability.
- Keep review evidence attached to the relevant product category and `group_sku`; do not create a standalone review-distribution sheet unless the user explicitly asks for it.
- Before finishing, verify the workbook was created and is readable. At minimum, check that the file exists, has nonzero size, opens via a spreadsheet reader, and contains the expected worksheets. When spreadsheet rendering tools are available, render or inspect representative sheets to catch blank sheets, clipped headers, and unreadable long-text ranges.
- If Excel generation fails because of missing local tooling, provide the Markdown source and clearly state that Excel rendering/export failed, including the blocker. Do not silently deliver only Markdown.

## Report Output

Use this structure as the report content model, then export it to an Excel workbook unless the user requests a different format:

```markdown
# 退货与评论分析报告

## 1. 分析口径
- 退货发生周期:
- 订单归属周期:
- 产品分类口径:
- SKU归类规则: 产品分类内取SKU前8个字符
- 数据范围:
- 匹配口径:
- 退货率口径:
- 窗口成熟度:
- Review使用口径: 仅作为退货备注/平台原因的辅助验证，不单独作为分析对象
- 备注说明:

> 结构说明: 从第2节开始，每个产品分类独立成章；每个品类内部按“分类概览 → 归类SKU表现 → 退货原因与SKU明细 → Review辅助判断 → 运营端与产品端行动清单”的固定框架展开。

## 2. 产品分类：XXXXX（P0/P1/P2/P3）
### 2.1 分类概览
- 分类结论:
- 主要退货问题:
- Review辅助覆盖:
- 亚马逊运营建议:
- 产品维度建议:
- 数据限制:

### 2.2 归类SKU表现
| 归类SKU | 退货数量/记录数 | 销量/订单量 | 退货率 | 主要退货原因 | 买家备注摘要 | Review辅助证据 | 优先级 |
|---|---:|---:|---:|---|---|---|---|

### 2.3 退货原因与SKU明细
#### SKU前8位: XXXXXXXX
- 退货概况:
- 买家退货备注主题:
- Review辅助验证:
- 可能根因:
- 证据强度:
- 亚马逊运营优化建议:
- 产品维度优化建议:

### 2.4 Review辅助判断
- Review仅用于辅助验证退货备注和平台退货原因，不单独作为问题分布或独立结论来源。
- Review辅助判断:
- 具体Review证据:

### 2.5 运营端与产品端行动清单
| 优先级 | 对象/SKU | 问题 | 证据来源 | 运营端可落地动作 | 产品端可落地动作 | 验收/下一步 |
|---|---|---|---|---|---|---|

## 3. 产品分类：YYYYY（P0/P1/P2/P3）
### 3.1 分类概览
...
### 3.5 运营端与产品端行动清单
...

## 附录：数据限制与下一步
- 不展示全局主题分布:
- Review数据限制:
- 销量/订单量限制:
- 下次需要补充的数据:
```

Number category sections sequentially starting from `## 2`. Within each product category, use matching subsection numbers such as `### 2.1` through `### 2.5`, then `### 3.1` through `### 3.5` for the next category. Do not create a global `按产品分类总览`, global `产品分类详细分析`, or global `按产品分类行动清单` section.

After writing the source report, convert it into an Excel workbook and return the `.xlsx` path in the final response. Mention the Markdown source only as a secondary artifact.

Recommended workbook mapping:
- `目录` sheet: section index, product category, priority level, worksheet name, and short notes.
- `分析口径` sheet: all items from section 1.
- Product-category sheets: one sheet per `## 产品分类`, preserving the same `x.1` through `x.5` internal framework.
- SKU and action tables: keep as real editable worksheet tables/ranges, not screenshots or pasted images.

If sales/order data is missing or too large to collect, keep the output structure but replace return-rate fields with `未提供或暂不使用销量/订单量，无法计算真实退货率`; continue ranking by return concentration, reason severity, and review corroboration. If review data is missing, replace review fields with `未提供Review数据，无法交叉验证`.

## Analysis Rules

- Do not overstate causality. Mark uncertain conclusions as `推测` or `需进一步验证`.
- If buyer return remarks are sparse, rely more on platform return reason fields and state the limitation.
- Sales/order data is useful but not mandatory. If absent, too large, or historically messy, rank by return quantity, reason severity, inventory disposition, and review corroboration; explicitly state that true return rate cannot be judged.
- If sales/order data is present, do not equate high return count with high return-rate risk until the denominator and order-month attribution are checked.
- Do not use same-month sales as the default denominator for a monthly return report when order date exists. Monthly return reports usually contain returns from earlier order months.
- Keep return-processing volume and order-attributed return rate separate in the narrative.
- Flag immature order months and avoid making strong product severity claims from incomplete return windows.
- Flag very old order dates as historical/exception returns. Do not force the user to download huge historical sales/order files for these rows unless exact rate analysis is required.
- If review data is present, use reviews to validate recurring product and expectation issues, but do not treat reviews as a perfect proxy for returns because review writers are a biased subset of buyers.
- Review is supporting evidence only. Do not lead with review distributions or create a standalone review analysis unless requested; attach review evidence to the relevant product category and `group_sku`.
- Do not let reviews create new analytical objects. If a review SKU is absent from the return report, do not include it in the product-category tables or priority list.
- If review themes conflict with return themes, call out the mismatch and suggest further checks such as customer service messages, QA tickets, or child-SKU drilldown.
- Do not present global return-theme distributions as the main analytical structure when product categories differ. Summarize themes inside each product category to keep conclusions actionable.
- Do not use a global 2-5 report structure that contains all categories inside shared `总览/明细/行动清单` sections. From section 2 onward, each product category must be a self-contained mini-report with its own 1-5 subframework.
- Do not present the final action list as one mixed cross-category table. Put each category's action list inside that category's own section so each category owner can execute without filtering unrelated products.
- If multiple languages appear in comments or reviews, summarize in Chinese and retain important original phrases only when helpful.
- Use return quantity when available; otherwise use record count.
- Prioritize recommendations by return rate, return concentration, repeated complaint specificity, review corroboration, and whether the issue can be fixed quickly in listing operations.
- Keep suggestions actionable and specific to the observed reason, not generic marketplace advice. Every recommendation must identify the operating surface to change or the product/QC test to perform.
- Separate every action recommendation into `运营端` and `产品端`; do not combine them in a single generic "建议动作" column.
- Deliver Excel (`.xlsx`) as the default final format. Markdown is a working source, not the final deliverable, unless the user asks for Markdown.

## Example Triggers

- `帮我分析下这份退货报告`
- `帮我分析这份退货报告并输出Excel`
- `帮我分析这份退货和Review，并直接给我Excel表`
- `按SKU前8位汇总退货原因，并结合销量算退货率`
- `按订购时间把退货归因到订单月份，再匹配销量计算退货率`
- `先不使用销量表，结合退货报告和Review判断主要问题`
- `结合退货报告、销量表和Review，判断哪些SKU最需要优化`
- `分析买家退货备注和差评，分别从亚马逊运营和产品角度给建议`

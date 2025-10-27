# ServiceNow Scripts (Scoped & Safe)

A curated set of ServiceNow utilities showcasing scoped Glide APIs, platform-safe patterns, and readable, reusable code.

## Feature: Safely Inspect a Table’s Metadata

> :warning: **Requires ECMAScript 6**  
> Background scripts in the **global scope** run in ES5 and will throw syntax errors.  
> Run this in a **custom application scope**, which supports ES6 (let/const, arrow functions, etc.).


Builds a complete **column data model** for any table using only **scoped** Glide APIs — no direct queries to system tables and no inserts.

### What it does
- Spins up an in-memory **`GlideRecordSecure` scratchpad** to read field metadata safely.
- Returns a structured model per column: label, internal type, max length, mandatory, inheritance source, auto/sys_id, virtual.
- **References:** includes target table, display field, and class label.
- **Choices:** enumerates display labels and values in order (skips `sys_class_name` to avoid table model morphing).
- Produces a JSON-like object suitable for docs, generators, or integration mapping.

**Great for:** dynamic record producers and record operations, schema docs, integration design where sys_* tables are off-limits.

<details>
<summary>Sample Background Script</summary>

```javascript
/**
 * Builds a detailed data model for a specified table using only scoped Glide APIs.
 * Initializes an in-memory GlideRecordSecure instance to safely inspect field metadata without querying system tables or inserting data.
 *
 * @param {string} pTable - The name of the table to describe (e.g., 'incident').
 * @returns {Object} An object containing metadata for each column, including label, type, length, mandatory status, inheritance, and choice/reference info.
 *
 * Example:
 * const model = getTableDataModelEcma6('incident');
 * gs.info(JSON.stringify(model, null, '\t'));
 */

const model = getTableDataModelEcma6('incident');
gs.info(JSON.stringify(model, null, '\t'));

function getTableDataModelEcma6(pTable) {

    try {

        if (!pTable) return;

        let columnDataModel = [];

        // Create a temporary GlideRecord instance (scratchpad). Used to resolve display values and field metadata without inserting data.
        const grScratchpad = new GlideRecordSecure(pTable);
        if (!grScratchpad.isValid()) return;
        grScratchpad.initialize();

        // Get a list of column names from the scratchpad.
        const columnNames = Array.from(grScratchpad.getElements() || [], pColumn => pColumn.getName()).sort();


        // Retrieve each scratchpad column's metadata and design specification.
        for (const columnName of columnNames) {

            // The scratchpad's writeable column.
            const column = grScratchpad[columnName];

            // The scratchpad column's metadata, such as its choices, its reference table, its source table, and so on.
            const columnMetadata = grScratchpad.getElement(columnName);

            // The scratchpad column's design specification, such as its internal type, its length, whether it's mandatory, and so on.
            const columnSpec = columnMetadata.getED();

            const internalType = columnSpec.getInternalType();
            const referenceDataModel = {};
            const choiceDataModel = [];
            const choicesAsJava = columnMetadata.getChoices() || [];

            // If the scratchpad column is a reference, then record the reference-specific details.
            if (internalType === 'reference') {
                const reference = columnMetadata.getReferenceTable();
                const grReferenceDataModel = new GlideRecordSecure(reference);
                grReferenceDataModel.initialize();
                referenceDataModel.label = grReferenceDataModel.getClassDisplayValue();
                referenceDataModel.value = reference;
                referenceDataModel.display_field = grReferenceDataModel.getDisplayName();
            }

            // If the scratchpad column comprises choice values and it is not sys_class_name, then record the choice-specific details.
            // sys_class_name is excluded because setting it to a value that is different from the current table changes the table itself, destroying the table data model.
            else if (choicesAsJava.length > 0 && columnName !== 'sys_class_name') {

                // Iterate through each of the column's possible values.
                Array.from(choicesAsJava).forEach((pChoice, pIndex) => {

                    // Set the column's value, then retrieve the label of the value.
                    const value = internalType === 'integer' ? Number(pChoice) : String(pChoice);
                    column.setValue(value);
                    choiceDataModel.push({
                        label: column.getDisplayValue(),
                        value: value,
                        order: pIndex + 1
                    });
                });
            }

            // If the scratchpad column is sys_class_name, then record the choice-specific detail.
            else if (columnName === 'sys_class_name') {
                choiceDataModel.push({
                    label: grScratchpad.getClassDisplayValue(),
                    value: pTable,
                    order: 1
                });
            }

            // Record the scratchpad column's metadata and design specificaiton to the table's data model.
            columnDataModel.push({
                column_name: columnName,
                column_label: columnSpec.getLabel(),
                internal_type: internalType,
                max_length: columnSpec.getLength(),
                mandatory: columnSpec.isMandatory(),
                inherited: columnMetadata.getTableName() !== pTable,
                auto: columnSpec.isAutoOrSysID(),
                virtual: columnSpec.isVirtual(),
                reference: referenceDataModel,
                choices: choiceDataModel
            });
        }

        // Server-callable can return an object.
        return {
            tableDataModel: columnDataModel
        };
    } catch (err) {
        return String(err);
    }

}
```
</details>

<details>
<summary>Sample Response (for incident)</summary>

```json
{
	"tableDataModel": [
		{
			"column_name": "active",
			"column_label": "Active",
			"internal_type": "boolean",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "activity_due",
			"column_label": "Activity due",
			"internal_type": "due_date",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "additional_assignee_list",
			"column_label": "Additional assignee list",
			"internal_type": "glide_list",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "approval",
			"column_label": "Approval",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Not Yet Requested",
					"value": "not requested",
					"order": 1
				},
				{
					"label": "Requested",
					"value": "requested",
					"order": 2
				},
				{
					"label": "Approved",
					"value": "approved",
					"order": 3
				},
				{
					"label": "Rejected",
					"value": "rejected",
					"order": 4
				}
			]
		},
		{
			"column_name": "approval_history",
			"column_label": "Approval history",
			"internal_type": "journal",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "approval_set",
			"column_label": "Approval set",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "assigned_to",
			"column_label": "Assigned to",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "assignment_group",
			"column_label": "Assignment group",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Group",
				"value": "sys_user_group",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "business_duration",
			"column_label": "Business duration",
			"internal_type": "glide_duration",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "business_impact",
			"column_label": "Business impact",
			"internal_type": "string",
			"max_length": 4000,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "business_service",
			"column_label": "Service",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Service",
				"value": "cmdb_ci_service",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "business_stc",
			"column_label": "Business resolve time",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "calendar_duration",
			"column_label": "Duration",
			"internal_type": "glide_duration",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "calendar_stc",
			"column_label": "Resolve time",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "caller_id",
			"column_label": "Caller",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "category",
			"column_label": "Category",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Inquiry / Help",
					"value": "inquiry",
					"order": 1
				},
				{
					"label": "Software",
					"value": "software",
					"order": 2
				},
				{
					"label": "Hardware",
					"value": "hardware",
					"order": 3
				},
				{
					"label": "Network",
					"value": "network",
					"order": 4
				},
				{
					"label": "Database",
					"value": "database",
					"order": 5
				},
				{
					"label": "sFone",
					"value": "sFone",
					"order": 6
				}
			]
		},
		{
			"column_name": "cause",
			"column_label": "Probable cause",
			"internal_type": "string",
			"max_length": 4000,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "caused_by",
			"column_label": "Caused by Change",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Change Request",
				"value": "change_request",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "child_incidents",
			"column_label": "Child Incidents",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "close_code",
			"column_label": "Resolution code",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Duplicate",
					"value": "Duplicate",
					"order": 1
				},
				{
					"label": "Known error",
					"value": "Known error",
					"order": 2
				},
				{
					"label": "No resolution provided",
					"value": "No resolution provided",
					"order": 3
				},
				{
					"label": "Resolved by caller",
					"value": "Resolved by caller",
					"order": 4
				},
				{
					"label": "Resolved by change",
					"value": "Resolved by change",
					"order": 5
				},
				{
					"label": "Resolved by problem",
					"value": "Resolved by problem",
					"order": 6
				},
				{
					"label": "Resolved by request",
					"value": "Resolved by request",
					"order": 7
				},
				{
					"label": "Solution provided",
					"value": "Solution provided",
					"order": 8
				},
				{
					"label": "Workaround provided",
					"value": "Workaround provided",
					"order": 9
				},
				{
					"label": "User error",
					"value": "User error",
					"order": 10
				}
			]
		},
		{
			"column_name": "close_notes",
			"column_label": "Close notes",
			"internal_type": "string",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "closed_at",
			"column_label": "Closed",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "closed_by",
			"column_label": "Closed by",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "cmdb_ci",
			"column_label": "Configuration item",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Configuration Item",
				"value": "cmdb_ci",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "comments",
			"column_label": "Additional comments",
			"internal_type": "journal_input",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "comments_and_work_notes",
			"column_label": "Comments and Work notes",
			"internal_type": "journal_list",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "company",
			"column_label": "Company",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": true,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Company",
				"value": "core_company",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "contact_type",
			"column_label": "Contact type",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Email",
					"value": "email",
					"order": 1
				},
				{
					"label": "Phone",
					"value": "phone",
					"order": 2
				},
				{
					"label": "Self-service",
					"value": "self-service",
					"order": 3
				},
				{
					"label": "Virtual Agent",
					"value": "virtual_agent",
					"order": 4
				},
				{
					"label": "Walk-in",
					"value": "walk-in",
					"order": 5
				}
			]
		},
		{
			"column_name": "contract",
			"column_label": "Contract",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Contract",
				"value": "ast_contract",
				"display_field": "vendor_contract"
			},
			"choices": []
		},
		{
			"column_name": "correlation_display",
			"column_label": "Correlation display",
			"internal_type": "string",
			"max_length": 100,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "correlation_id",
			"column_label": "Correlation ID",
			"internal_type": "string",
			"max_length": 100,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "delivery_plan",
			"column_label": "Delivery plan",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Execution Plan",
				"value": "sc_cat_item_delivery_plan",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "delivery_task",
			"column_label": "Delivery task",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Execution Plan Task",
				"value": "sc_cat_item_delivery_task",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "description",
			"column_label": "Description",
			"internal_type": "string",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "due_date",
			"column_label": "Due date",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "escalation",
			"column_label": "Escalation",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Normal",
					"value": 0,
					"order": 1
				},
				{
					"label": "Moderate",
					"value": 1,
					"order": 2
				},
				{
					"label": "High",
					"value": 2,
					"order": 3
				},
				{
					"label": "Overdue",
					"value": 3,
					"order": 4
				}
			]
		},
		{
			"column_name": "expected_start",
			"column_label": "Expected start",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "follow_up",
			"column_label": "Follow up",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "group_list",
			"column_label": "Group list",
			"internal_type": "glide_list",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "hold_reason",
			"column_label": "On hold reason",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Awaiting Caller",
					"value": 1,
					"order": 1
				},
				{
					"label": "Awaiting Change",
					"value": 5,
					"order": 2
				},
				{
					"label": "Awaiting Problem",
					"value": 3,
					"order": 3
				},
				{
					"label": "Awaiting Vendor",
					"value": 4,
					"order": 4
				}
			]
		},
		{
			"column_name": "impact",
			"column_label": "Impact",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "1 - High",
					"value": 1,
					"order": 1
				},
				{
					"label": "2 - Medium",
					"value": 2,
					"order": 2
				},
				{
					"label": "3 - Low",
					"value": 3,
					"order": 3
				}
			]
		},
		{
			"column_name": "incident_state",
			"column_label": "Incident state",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "New",
					"value": 1,
					"order": 1
				},
				{
					"label": "In Progress",
					"value": 2,
					"order": 2
				},
				{
					"label": "On Hold",
					"value": 3,
					"order": 3
				},
				{
					"label": "Resolved",
					"value": 6,
					"order": 4
				},
				{
					"label": "Closed",
					"value": 7,
					"order": 5
				},
				{
					"label": "Canceled",
					"value": 8,
					"order": 6
				}
			]
		},
		{
			"column_name": "knowledge",
			"column_label": "Knowledge",
			"internal_type": "boolean",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "location",
			"column_label": "Location",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Location",
				"value": "cmn_location",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "made_sla",
			"column_label": "Made SLA",
			"internal_type": "boolean",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "notify",
			"column_label": "Notify",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Do Not Notify",
					"value": 1,
					"order": 1
				},
				{
					"label": "Send Email",
					"value": 2,
					"order": 2
				},
				{
					"label": "Telephone",
					"value": 3,
					"order": 3
				}
			]
		},
		{
			"column_name": "number",
			"column_label": "Number",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "opened_at",
			"column_label": "Opened",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "opened_by",
			"column_label": "Opened by",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "order",
			"column_label": "Order",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "origin_id",
			"column_label": "Origin",
			"internal_type": "document_id",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "origin_table",
			"column_label": "Origin table",
			"internal_type": "table_name",
			"max_length": 80,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "parent",
			"column_label": "Parent",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Task",
				"value": "task",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "parent_incident",
			"column_label": "Parent Incident",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Incident",
				"value": "incident",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "priority",
			"column_label": "Priority",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "1 - Critical",
					"value": 1,
					"order": 1
				},
				{
					"label": "2 - High",
					"value": 2,
					"order": 2
				},
				{
					"label": "3 - Moderate",
					"value": 3,
					"order": 3
				},
				{
					"label": "4 - Low",
					"value": 4,
					"order": 4
				},
				{
					"label": "5 - Planning",
					"value": 5,
					"order": 5
				}
			]
		},
		{
			"column_name": "problem_id",
			"column_label": "Problem",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Problem",
				"value": "problem",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "reassignment_count",
			"column_label": "Reassignment count",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "rejection_goto",
			"column_label": "Rejection goto",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Task",
				"value": "task",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "reopen_count",
			"column_label": "Reopen count",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "reopened_by",
			"column_label": "Last reopened by",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "reopened_time",
			"column_label": "Last reopened at",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "resolved_at",
			"column_label": "Resolved",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "resolved_by",
			"column_label": "Resolved by",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "rfc",
			"column_label": "Change Request",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Change Request",
				"value": "change_request",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "route_reason",
			"column_label": "Transfer reason",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Transfer with Resolution",
					"value": 1,
					"order": 1
				},
				{
					"label": "Transfer without Resolution",
					"value": 2,
					"order": 2
				}
			]
		},
		{
			"column_name": "service_offering",
			"column_label": "Service offering",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Offering",
				"value": "service_offering",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "severity",
			"column_label": "Severity",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "1 - High",
					"value": 1,
					"order": 1
				},
				{
					"label": "2 - Medium",
					"value": 2,
					"order": 2
				},
				{
					"label": "3 - Low",
					"value": 3,
					"order": 3
				}
			]
		},
		{
			"column_name": "short_description",
			"column_label": "Short description",
			"internal_type": "string",
			"max_length": 160,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Issue with a web page",
					"value": "Issue with a web page",
					"order": 1
				},
				{
					"label": "Issue with email",
					"value": "Issue with email",
					"order": 2
				},
				{
					"label": "Issue with networking",
					"value": "Issue with networking",
					"order": 3
				},
				{
					"label": "New employee hire",
					"value": "New employee hire",
					"order": 4
				},
				{
					"label": "Request for a Blackberry",
					"value": "Request for a Blackberry",
					"order": 5
				},
				{
					"label": "Request for a new service",
					"value": "Request for a new service",
					"order": 6
				},
				{
					"label": "Request for help",
					"value": "Request for help",
					"order": 7
				},
				{
					"label": "Request for new or upgraded hardware",
					"value": "Request for new or upgraded hardware",
					"order": 8
				},
				{
					"label": "Request for new or upgraded software",
					"value": "Request for new or upgraded software",
					"order": 9
				},
				{
					"label": "Reset my password",
					"value": "Reset my password",
					"order": 10
				}
			]
		},
		{
			"column_name": "sla_due",
			"column_label": "SLA due",
			"internal_type": "due_date",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "state",
			"column_label": "State",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "(-5)",
					"value": -5,
					"order": 1
				},
				{
					"label": "New",
					"value": 1,
					"order": 2
				},
				{
					"label": "In Progress",
					"value": 2,
					"order": 3
				},
				{
					"label": "On Hold",
					"value": 3,
					"order": 4
				},
				{
					"label": "(4)",
					"value": 4,
					"order": 5
				},
				{
					"label": "Closed",
					"value": 7,
					"order": 6
				}
			]
		},
		{
			"column_name": "subcategory",
			"column_label": "Subcategory",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Antivirus",
					"value": "antivirus",
					"order": 1
				},
				{
					"label": "CPU",
					"value": "cpu",
					"order": 2
				},
				{
					"label": "DB2",
					"value": "db2",
					"order": 3
				},
				{
					"label": "DHCP",
					"value": "dhcp",
					"order": 4
				},
				{
					"label": "Disk",
					"value": "disk",
					"order": 5
				},
				{
					"label": "DNS",
					"value": "dns",
					"order": 6
				},
				{
					"label": "Email",
					"value": "email",
					"order": 7
				},
				{
					"label": "Internal Application",
					"value": "internal application",
					"order": 8
				},
				{
					"label": "IP Address",
					"value": "ip address",
					"order": 9
				},
				{
					"label": "Keyboard",
					"value": "keyboard",
					"order": 10
				},
				{
					"label": "Memory",
					"value": "memory",
					"order": 11
				},
				{
					"label": "Monitor",
					"value": "monitor",
					"order": 12
				},
				{
					"label": "Mouse",
					"value": "mouse",
					"order": 13
				},
				{
					"label": "MS SQL Server",
					"value": "sql server",
					"order": 14
				},
				{
					"label": "Operating System",
					"value": "os",
					"order": 15
				},
				{
					"label": "Oracle",
					"value": "oracle",
					"order": 16
				},
				{
					"label": "VPN",
					"value": "vpn",
					"order": 17
				},
				{
					"label": "Wireless",
					"value": "wireless",
					"order": 18
				}
			]
		},
		{
			"column_name": "sys_class_name",
			"column_label": "Task type",
			"internal_type": "sys_class_name",
			"max_length": 80,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Incident",
					"value": "incident",
					"order": 1
				}
			]
		},
		{
			"column_name": "sys_created_by",
			"column_label": "Created by",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_created_on",
			"column_label": "Created",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_domain",
			"column_label": "Domain",
			"internal_type": "domain_id",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_domain_path",
			"column_label": "Domain Path",
			"internal_type": "domain_path",
			"max_length": 255,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_id",
			"column_label": "Sys ID",
			"internal_type": "GUID",
			"max_length": 32,
			"mandatory": true,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_mod_count",
			"column_label": "Updates",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_tags",
			"column_label": "Tags",
			"internal_type": "related_tags",
			"max_length": 0,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_updated_by",
			"column_label": "Updated by",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "sys_updated_on",
			"column_label": "Updated",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": true,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "task_effective_number",
			"column_label": "Effective number",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": true,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "task_for",
			"column_label": "Task for",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "User",
				"value": "sys_user",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "time_worked",
			"column_label": "Time worked",
			"internal_type": "timer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "u_sfone_model",
			"column_label": "sFone Model",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": false,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "universal_request",
			"column_label": "Universal Request",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Task",
				"value": "task",
				"display_field": "number"
			},
			"choices": []
		},
		{
			"column_name": "upon_approval",
			"column_label": "Upon approval",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Proceed to Next Task",
					"value": "proceed",
					"order": 1
				},
				{
					"label": "Wait for a User to Close this task",
					"value": "do_nothing",
					"order": 2
				}
			]
		},
		{
			"column_name": "upon_reject",
			"column_label": "Upon reject",
			"internal_type": "string",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "Cancel all future Tasks",
					"value": "cancel",
					"order": 1
				},
				{
					"label": "Go to a previous Task",
					"value": "goto",
					"order": 2
				}
			]
		},
		{
			"column_name": "urgency",
			"column_label": "Urgency",
			"internal_type": "integer",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": [
				{
					"label": "1 - High",
					"value": 1,
					"order": 1
				},
				{
					"label": "2 - Medium",
					"value": 2,
					"order": 2
				},
				{
					"label": "3 - Low",
					"value": 3,
					"order": 3
				}
			]
		},
		{
			"column_name": "user_input",
			"column_label": "User input",
			"internal_type": "user_input",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "variables",
			"column_label": "Variables",
			"internal_type": "variables",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "variables",
			"column_label": "Variables",
			"internal_type": "variables",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "variables",
			"column_label": "Variables",
			"internal_type": "variables",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "watch_list",
			"column_label": "Watch list",
			"internal_type": "glide_list",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "wf_activity",
			"column_label": "Workflow activity",
			"internal_type": "reference",
			"max_length": 32,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {
				"label": "Workflow Activity",
				"value": "wf_activity",
				"display_field": "name"
			},
			"choices": []
		},
		{
			"column_name": "work_end",
			"column_label": "Actual end",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "work_notes",
			"column_label": "Work notes",
			"internal_type": "journal_input",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "work_notes_list",
			"column_label": "Work notes list",
			"internal_type": "glide_list",
			"max_length": 4000,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		},
		{
			"column_name": "work_start",
			"column_label": "Actual start",
			"internal_type": "glide_date_time",
			"max_length": 40,
			"mandatory": false,
			"inherited": true,
			"auto": false,
			"virtual": false,
			"reference": {},
			"choices": []
		}
	]
}

```
</details>

# Datagrid Inline Row Editing

Allowing users to conveniently and swiftly edit selected DataGrid rows

https://github.com/stadium-software/datagrid-inline-row-edit/assets/2085324/88135464-9da0-4155-939b-bb9e31bf3931

## Sample applications
This repo contains one Stadium 6.7 application
[DataGridInlineRowEditing.sapz](Stadium6/DataGridInlineRowEditing.sapz?raw=true)

# Content
- [Datagrid Inline Row Editing](#datagrid-inline-row-editing)
  - [Sample applications](#sample-applications)
- [Content](#content)
  - [Version](#version)
- [Setup](#setup)
  - [Application Setup](#application-setup)
  - [Database, Connector and DataGrid](#database-connector-and-datagrid)
  - [Type Setup](#type-setup)
  - [Global Script Setup](#global-script-setup)
  - [Page-Script Setup](#page-script-setup)
  - [Page Setup](#page-setup)
  - [Page.Load Event Setup](#pageload-event-setup)
  - [Edit.Click Event Setup](#editclick-event-setup)
- [Styling](#styling)
  - [Applying the CSS](#applying-the-css)
  - [Customising CSS](#customising-css)
  - [CSS Upgrading](#css-upgrading)

## Version 
1.4

1.1 Row Editing

1. Added code to skip rules that do not have corresponding columns
2. Bug fix: Empty column headings caused save button to appear in the wrong column
3. Added code to detect uniqueness of DataGrid class on page

1.2 Updated script to cater for changed DataGrid rendering

1.3 Added custom event handler feature

1.4 Fixed Selectable column bug

1.5 Enabled adding IDColumn and EditColumn as column numbers

# Setup

## Application Setup
1. Check the *Enable Style Sheet* checkbox in the application properties

## Database, Connector and DataGrid
1. Use the instructions from [this repo](https://github.com/stadium-software/samples-database) to setup the database and DataGrid for this sample
2. The DataGrid must contain an Edit column and the Edit column must have a click event handler

## Type Setup
1. Create a *Type* called "FormField"
2. Add the following properties to the type
   1. "name" (Any)
   2. "type" (Any)
   3. "required" (Any)
   4. "min" (Any)
   5. "max" (Any)
   6. "data" (List)
      1. "Item" (Any)

![Form Fields](images/FormFieldType.png)

## Global Script Setup
1. Create a Global Script called "EditableRow"
2. Add five input parameters to the Global Script
   1. DataGridClass
   2. EditColumnHeader
   3. FormFields
   4. IdentityColumnHeader
   5. IdentityValue
   6. CallbackScript
3. Drag a *JavaScript* action into the script
4. Add the Javascript below into the JavaScript code property
```javascript
/*Stadium Script Version 1.5*/
let scope = this;
let callback = ~.Parameters.Input.CallbackScript;
let dgClassName = "." + ~.Parameters.Input.DataGridClass;
let dg = document.querySelectorAll(dgClassName);
if (dg.length == 0) {
    dg = document.querySelector(".data-grid-container");
} else if (dg.length > 1) {
    console.error("The class '" + dgClassName + "' is assigned to multiple DataGrids. DataGrids using this script must have unique classnames");
    return false;
} else { 
    dg = dg[0];
}
dg.classList.add("stadium-inline-edit-datagrid");
let table = dg.querySelector("table");
let rowFormFields = ~.Parameters.Input.FormFields;
let IDColumn = ~.Parameters.Input.IdentityColumnHeader;
let IDValue = ~.Parameters.Input.IdentityValue;
let EditColumn = ~.Parameters.Input.EditColumnHeader;
let IDColNo, rowNumber;
let options = {
    characterData: true,
    attributes: false,
    childList: true,
    subtree: true,
    characterDataOldValue: true,
},
observer = new MutationObserver(resetDataGrid);
let getIndex = (heystack, needle) => {
    let result = needle - 1;
    if (isNaN(parseFloat(needle))) { 
        result = heystack.findIndex((col) => col.name == needle);
    }
    return result;
};

insertForm();
initForm();
document.onkeydown = function (evt) {
    evt = evt || window.event;
    var isEscape = false;
    if ("key" in evt) {
        isEscape = (evt.key === "Escape" || evt.key === "Esc");
    } else {
        isEscape = (evt.keyCode === 27);
    }
    if (isEscape) {
        resetDataGrid();
    }
};
observer.observe(dg, options);

/*--------------------------------------------------------------------------------------*/

function initForm() { 
    let enrichedRowData = enrichRowData(rowFormFields);
    let result = enrichedRowData.find((obj) => {
        return obj.type == "Identity";
    });
    if (!result) {
        console.error("The identity column was not found");
        return false;
    }
    IDColNo = result.colNo;

    let IDCells = table.querySelectorAll("tbody tr td:nth-child(" + IDColNo + ")");
    for (let i = 0; i < IDCells.length; i++) {
        let rowtr = IDCells[i].parentElement;
        if (IDCells[i].innerText == IDValue) {
            rowNumber = i + 1;
        } else { 
            rowtr.classList.add("opacity");
            rowtr.addEventListener("click", resetDataGrid);
        }
    }

    let row = table.querySelector("tbody tr:nth-child(" + rowNumber + ")");
    row.classList.add("edit-orig");
    let editform = document.createElement("tr");
    editform.classList.add("edit-form");
    for (let i = 0; i < enrichedRowData.length; i++) {
        let origCell = row.querySelectorAll("td")[i];
        let origStyles = origCell.getAttribute("style");
        let cell = document.createElement("td");
        cell.setAttribute("style", origStyles);
        let name = enrichedRowData[i].name;
        let value = origCell.textContent;
        let type = enrichedRowData[i].type;
        let data = enrichedRowData[i].data;
        let min = enrichedRowData[i].min;
        let max = enrichedRowData[i].max;
        let required = enrichedRowData[i].required;
        let el;
        if (type == "date") {
            el = document.createElement("input");
            el.setAttribute("type", "date");
            el.classList.add("form-control");
            if (min) {
                let dmin = new Date(min);
                min = dmin.getFullYear() + '-' + ('0' + (dmin.getMonth() + 1)).slice(-2) + '-' + ('0' + dmin.getDate()).slice(-2);
                el.setAttribute("min", min);
            }
            if (max) {
                let dmax = new Date(max);
                max = dmax.getFullYear() + '-' + ('0' + (dmax.getMonth() + 1)).slice(-2) + '-' + ('0' + dmax.getDate()).slice(-2);
                el.setAttribute("max", max);
            }
            el.setAttribute("stadium-form-name", name);
            let d = new Date(value);
            el.value = d.getFullYear() + '-' + ('0' + (d.getMonth() + 1)).slice(-2) + '-' + ('0' + d.getDate()).slice(-2);
        } else if (type == "number") {
            el = document.createElement("input");
            el.setAttribute("type", "number");
            if (min) el.setAttribute("min", min);
            if (max) el.setAttribute("max", max);
            el.setAttribute("onkeydown", "return event.keyCode !== 69");
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } else if (type == "checkbox") {
            el = document.createElement("input");
            el.setAttribute("stadium-form-name", name);
            el.setAttribute("type", "checkbox");
            if (value == "true" || value == "Yes" || value == "1") {
                el.setAttribute("checked", "");
            }
        } else if (type == "dropdown") {
            el = document.createElement("select");
            for (let j = 0; j < data.length; j++) {
                let option = document.createElement("option");
                option.text = data[j];
                option.value = data[j];
                el.appendChild(option);
            }
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } else if (type == "EditLink") {
            el = document.createElement("button");
            el.classList.add("stadium-form-save");
            el.setAttribute("type", "submit");
            el.innerText = "Save";
        } else if (type == "text") {
            el = document.createElement("input");
            el.value = value;
            el.setAttribute("stadium-form-name", name);
            el.classList.add("form-control");
        } else { 
            if (!origCell.querySelector("button")) { 
                cell.textContent = value;
                el = document.createElement("input");
                el.value = value;
                el.setAttribute("stadium-form-name", name);
                el.setAttribute("type", "hidden");
            }
        }
        if (el) {
            el.classList.add("stadium-inline-form-control");
            if (required) el.setAttribute("required", "");
            cell.appendChild(el);
        }
        editform.appendChild(cell);
    }
    insertAfter(editform, row);
}
function enrichRowData(data) {
    let arrHeadings = table.querySelectorAll("thead th");
    data.forEach((element, index) => {
        if (element.name) {
            data[index].name = element.name.toLowerCase().replaceAll(" ", "");
        } else { 
            element.name = "";
        }
    });
    for (let i = 0; i < arrHeadings.length; i++) {
        let heading = arrHeadings[i].innerText.toLowerCase().replaceAll(" ", "");
        let index = getIndex(data, heading);
        if (index > -1) {
            data[index].colNo = i + 1;
        } else if (IDColumn.toString().toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
            data.push({ name: IDColumn, colNo: i + 1, type: "Identity" });
        } else if (EditColumn.toString().toLowerCase() == arrHeadings[i].innerText.toLowerCase()) {
            data.push({ name: EditColumn, colNo: i + 1, type: "EditLink" });
        } else if (IDColumn == (i + 1)) {
            data.push({ name: "Identity", colNo: IDColumn, type: "Identity" });
        } else if (EditColumn == (i + 1)) {
            data.push({ name: "", colNo: EditColumn, type: "EditLink" });
        } else { 
            data.push({colNo: i + 1, name: heading});
        }
    }
    data.sort((a, b) => a.colNo - b.colNo);
    return data;
}
function resetDataGrid(){ 
    observer.disconnect();
    let editorig = table.querySelector(".edit-orig");
    if (editorig) editorig.classList.remove("edit-orig");
    let editform = table.querySelector(".edit-form");
    if (editform) editform.remove();
    let opaque = table.querySelectorAll(".opacity");
    for (let i = 0; i < opaque.length; i++) {
        opaque[i].classList.remove("opacity");
        opaque[i].removeEventListener("click", resetDataGrid);
    }
}
async function saveButtonClick(e) { 
    e.preventDefault();
    let form = e.target.closest("form");
    let formFields = form.querySelectorAll("[stadium-form-name]");
    let arrData = [];
    for (let i = 0; i < formFields.length; i++) { 
        let fieldValue = formFields[i].value;
        if (formFields[i].getAttribute("type") == "checkbox") fieldValue = formFields[i].checked;
        let field = { Name: formFields[i].getAttribute("stadium-form-name"), Value: fieldValue};
        arrData.push(field);
    }
    await scope[callback](arrData);
    resetDataGrid();
}
function insertForm() { 
    let exists = false;
    let forms = document.querySelectorAll(".datagrid-inline-edit-form");
    for (let i = 0; i < forms.length; i++) { 
        if (dg.parentNode == forms[i]) { 
            exists = true;
        }
    }
    if (!exists) {
        let form = document.createElement('form');
        form.classList.add("datagrid-inline-edit-form");
        form.addEventListener("submit", saveButtonClick);
        insertAfter(form, dg);
        form.appendChild(dg);
    }
}
function insertAfter(newNode, existingNode) {
    existingNode.parentNode.insertBefore(newNode, existingNode.nextSibling);
}
```

## Page-Script Setup
1. Create a Script inside of the Page with any name you like (e.g. "SaveRow")
2. Add one input parameter to the Script
   1. RowData
3. Drag a *Notification* action into the script
4. In the *Message* property, select the *RowData* parameter from the *Script Input Parameters* category

## Page Setup
1. Drag a *DataGrid* control to the page ([see above](#database-connector-and-datagrid))
2. Add a class of your choosing to the *DataGrid* *Classes* property ti uniquely identify this DataGrid on this page (e.g datagrid-inline-edit)
3. Note: If multiple editable DataGrids are shown on one page, each DataGrid must have a unique classname

## Page.Load Event Setup
1. Populate the DataGrid with data ([see above](#database-connector-and-datagrid))

## Edit.Click Event Setup

DataGrid must contain an Edit column (a clickable row-level coilumn) and that column must have a click event handler

1. Drag a *List* action into the event script and name the List "FormFields"
2. Set the List *Item Type* property to "Types.FormField"
3. Define the editable columns of your datagrid and their form fields
   1. name: The heading of the column
   2. type: The type of the column. Supported are
      1. text
      2. date
      3. number
      4. checkbox
      5. dropdown
      6. email
   3. required: A boolean (add "true" if required)
   4. min: A minimum value for number or date columns
   5. max: A maximum value for number or date columns
   6. data: A simple list of values for dropdowns (see example below)
```json
= [{
	"name": "End Date",
	"type": "date",
	"required": "true"
},{
	"name": "First Name",
	"type": "text"
},{
	"name": "Last Name",
	"type": "text"
},{
	"name": "No of Children",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "No of Pets",
	"type": "number",
	"min": "0",
	"max": "10",
	"required": "true"
},{
	"name": "Start Date",
	"type": "date",
	"min": "01-01-2010",
	"max": "01-01-2024"
},{
	"name": "Healthy",
	"type": "checkbox"
},{
	"name": "Happy",
	"type": "checkbox"
},{
	"name": "Subscription",
	"type": "dropdown",
	"data": ["", "Subscribed", "Unsubscribed", "No data"],
	"required": "true"
}]
```
4. Drag the Global Script called "EditableRow" into the event script
5. Complete the Input properties for the script
   1. DataGridClass: The unique classname you assigned to the *DataGrid*
   2. EditColumnHeader: The header of the Edit column OR the column number (e.g. 1)
   3. FormFields: Select the *List* called "FormFields" from the dropdown
   4. IdentityColumnHeader: The header of the column that uniquely identifies each row (e.g. ID) OR the column number (e.g. 2)
   5. IdentityValue: The value from the IdentityColumn that uniquely identifies the row
   6. CallbackScript: The name of the page-level script that will process the updated data (e.g. SaveRow)

![Inline Editing Input Parameters](images/InlineRowEditingInputParameters.png)

# Styling
Various elements in this module can be styled using the two CSS files in this repo

## Applying the CSS

**Stadium 6.6 or higher**
1. Create a folder called "CSS" inside of your Embedded Files in your application
2. Drag the two CSS files from this repo [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) and [*datagrid-inline-edit.css*](datagrid-inline-edit.css) into that folder
3. Paste the link tags below into the *head* property of your application
```html
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit.css">
<link rel="stylesheet" href="{EmbeddedFiles}/CSS/datagrid-inline-edit-variables.css">
``` 

![](images/ApplicationHeadProp.png)

**Versions lower than 6.6**
1. Copy the CSS from the two css files into the Stylesheet in your application

## Customising CSS
1. Open the CSS file called [*datagrid-inline-edit-variables.css*](datagrid-inline-edit-variables.css) from this repo
2. Adjust the variables in the *:root* element as you see fit
3. Overwrite the file in the CSS folder of your application with the customised file

## CSS Upgrading
To upgrade the CSS in this module, follow the [steps outlined in this repo](https://github.com/stadium-software/samples-upgrading)

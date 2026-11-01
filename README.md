# Z_NLC_PASS_ORDER
Documentation for the `Z_NLC_PASS_ORDER` ABAP report, explaining PO and SES data retrieval, invoice and contract details, amount conversion and the execution of two Smart Forms to generate the NLC Pass Order.


# NLC Pass Order Generation – SAP ABAP Project

## 1. Project Overview

The `Z_NLC_PASS_ORDER` report is developed to generate an NLC Pass Order document using SAP Smart Forms.

The report retrieves information from different SAP MM tables based on the Purchase Order (PO) number and Service Entry Sheet (SES) number entered by the user.

It collects the following information:

- Purchase Order details.
- Service Entry Sheet details.
- Vendor information.
- LOA and contract details.
- Service period and amount.
- Invoice reference number and invoice date.
- Contract amount, including the applicable pricing condition.
- Invoice amount converted into words.

After retrieving the required information, the report fills the custom structure `ZST_NLC_PASS_ORDER` and passes it to two Smart Forms:

1. `ZFS_NLC_PASS_ORDER`
2. `ZFS_NLC_PASS_ORDER2`

Both Smart Forms are executed sequentially to generate the complete Pass Order output.

## 2. Technologies and SAP Objects Used

| Component         | Description                |
| ----------------- | -------------------------- |
| ABAP Report       | `Z_NLC_PASS_ORDER`         |
| Custom Structure  | `ZST_NLC_PASS_ORDER`       |
| Smart Form 1      | `ZFS_NLC_PASS_ORDER`       |
| Smart Form 2      | `ZFS_NLC_PASS_ORDER2`      |
| Function Module   | `SPELL_AMOUNT`             |
| Function Module   | `SSF_FUNCTION_MODULE_NAME` |
| Control Structure | `SSFCTRLOP`                |
| Output Structure  | `SSFCRESCL`                |

### SAP Tables Used

| Table | Description                  | Purpose                                           |
| ----- | ---------------------------- | ------------------------------------------------- |
| EKKO  | Purchasing Document Header   | Retrieves PO header information                   |
| EKPO  | Purchasing Document Item     | Retrieves PO item and subject                     |
| ESSR  | Service Entry Sheet Header   | Retrieves SES and service-period details          |
| LFA1  | Vendor Master General Data   | Retrieves vendor name                             |
| EKBE  | Purchasing Document History  | Identifies invoice-related PO history             |
| RBKP  | Invoice Document Header      | Retrieves invoice reference and date              |
| KONV  | Conditions (Classic Pricing) | Retrieves the selected contract pricing condition |

# 3. Input Parameters

The report contains three mandatory input parameters.

```
PARAMETERS:
  p_po_no  TYPE ekko-ebeln OBLIGATORY,
  p_ses_no TYPE essr-lblni OBLIGATORY,
  p_let_no TYPE zst_nlc_pass_order-zletter_no OBLIGATORY.
```

### Explanation

1\. P_PO_NO – Purchase Order Number

- Accepts the PO number from the user.
- Uses the standard SAP field `EKKO-EBELN`.
- Identifies the purchasing document whose details must be retrieved.

2\. P_SES_NO – Service Entry Sheet Number

- Accepts the SES number.
- Uses `ESSR-LBLNI`.
- Identifies the service entry sheet associated with the entered PO.

3\. P_LET_NO – Pass Order Letter Number

- Accepts the Pass Order letter number.
- Uses the custom structure field `ZST_NLC_PASS_ORDER-ZLETTER_NO`.
- This number is printed in the generated Pass Order.

`OBLIGATORY` ensures that all three parameters are entered before execution.

# 4. Data Declaration

## 4.1 Output Structure

```
DATA: gs_data TYPE zst_nlc_pass_order.
```

`GS_DATA` is the main output structure of the report.

It stores all the information collected from the SAP database tables.

The structure contains the information required by both Smart Forms, such as:

- Supplier name.
- Letter number and date.
- LOA number and date.
- Contract details.
- PO number.
- SES number.
- Service period.
- Amount.
- Invoice number and date.
- Amount in words.
- Contract amount.

The completed structure is passed to both Smart Forms through the interface parameter `IS_DATA`.

## 4.2 Database Work Areas

```
DATA: gs_ekko TYPE ekko,
      gs_ekpo TYPE ekpo,
      gs_lfa1 TYPE lfa1,
      gs_essr TYPE essr,
      gs_rbkp TYPE rbkp,
      gs_ekbe TYPE ekbe,
      gs_rseg TYPE rseg.

DATA: gt_essr TYPE STANDARD TABLE OF essr.
```

Each work area stores the selected record from its respective database table.

| Work Area | Table | Purpose                        |
| --------- | ----- | ------------------------------ |
| `GS_EKKO` | EKKO  | PO header details              |
| `GS_EKPO` | EKPO  | PO item details                |
| `GS_LFA1` | LFA1  | Vendor details                 |
| `GS_ESSR` | ESSR  | SES details                    |
| `GS_RBKP` | RBKP  | Invoice header work area       |
| `GS_EKBE` | EKBE  | PO history work area           |
| `GS_RSEG` | RSEG  | Invoice item work area         |
| `GT_ESSR` | ESSR  | Internal table for SES records |

## 4.3 Smart Form Function Module Variables

```
DATA: gv_fm_name1 TYPE rs38l_fnam,
      gv_fm_name2 TYPE rs38l_fnam.
```

Smart Forms generate their own function modules when activated.

These generated function module names are retrieved dynamically using `SSF_FUNCTION_MODULE_NAME`.

- `GV_FM_NAME1` stores the generated function module name of the first Smart Form.
- `GV_FM_NAME2` stores the generated function module name of the second Smart Form.

This avoids hardcoding generated function module names.

# 5. START-OF-SELECTION

```
START-OF-SELECTION.

  PERFORM get_data.
  PERFORM call_smartforms.
```

`START-OF-SELECTION` is the main processing event of the report.

Once the user executes the report, the following two FORM routines are called.

Step 1 – GET_DATA

Retrieves the required information from SAP database tables and fills `GS_DATA`.

Step 2 – CALL_SMARTFORMS

Executes the two Smart Forms and generates the Pass Order output.

The report first collects the data and then passes it to the Smart Forms.

# 6. GET_DATA – Retrieving Pass Order Information

The `GET_DATA` FORM is responsible for retrieving all the information required to generate the Pass Order.

It reads data from EKKO, ESSR, EKPO, LFA1, EKBE, RBKP, and KONV.

## 6.1 Retrieve Purchase Order Header Details – EKKO

```
SELECT SINGLE *
  FROM ekko
  INTO @gs_ekko
  WHERE ebeln = @p_po_no.

IF sy-subrc <> 0.
  MESSAGE 'PO number does not exist' TYPE 'E'.
ENDIF.
```

### Explanation

- `EKKO` is the standard SAP table that stores Purchasing Document Header information.
- `EBELN` represents the Purchase Order number.
- The entered PO number is compared with `EKKO-EBELN`.
- `SELECT SINGLE` retrieves the matching PO header record.
- The retrieved record is stored in `GS_EKKO`.

`SY-SUBRC` is checked to determine whether the record was found.

- `SY-SUBRC = 0`: PO record found.
- `SY-SUBRC <> 0`: PO record not found.

If the PO does not exist, the report displays:

`PO number does not exist`

The error message stops further processing.

Why is this validation required?

The PO is the main reference document for generating the Pass Order. All subsequent information is retrieved based on this PO.

## 6.2 Retrieve Service Entry Sheet Details – ESSR

```
SELECT SINGLE *
  FROM essr
  INTO @gs_essr
  WHERE lblni = @p_ses_no
    AND ebeln = @p_po_no.

APPEND gs_essr TO gt_essr.

IF sy-subrc <> 0.
  MESSAGE 'SES number does not belong to the entered PO'
          TYPE 'E'.
ENDIF.
```

### Explanation

- `ESSR` stores Service Entry Sheet header information.
- `LBLNI` represents the SES number.
- `EBELN` represents the associated PO number.

The report checks two conditions:

1. The entered SES number must exist.
2. The SES must belong to the entered PO.

The matching record is stored in `GS_ESSR`.

The selected SES is also appended to the internal table `GT_ESSR`.

If the SES does not belong to the entered PO, the report displays:

`SES number does not belong to the entered PO`

Important code correction:

The `APPEND` statement should be placed after checking whether the SELECT was successful. Otherwise, an initial work area can be appended when no SES is found.

Recommended arrangement:

```
IF sy-subrc = 0.
  APPEND gs_essr TO gt_essr.
ELSE.
  MESSAGE 'SES number does not belong to the entered PO'
          TYPE 'E'.
ENDIF.
```

If `GT_ESSR` is not used elsewhere, the internal table may not be necessary.

## 6.3 Retrieve PO Item Details – EKPO

```
SELECT SINGLE *
  FROM ekpo
  INTO @gs_ekpo
  WHERE ebeln = @p_po_no
    AND ebelp = @gs_essr-ebelp.

IF sy-subrc <> 0.
  MESSAGE 'PO item not found' TYPE 'E'.
ENDIF.
```

### Explanation

`EKPO` stores Purchase Order item-level information.

The report uses two fields:

- `EBELN`: Purchase Order number.
- `EBELP`: Purchase Order item number.

The item number is obtained from `GS_ESSR-EBELP`.

The report retrieves the corresponding PO item using the PO number and item number.

The selected record is stored in `GS_EKPO`.

The item description is later used to populate the Pass Order subject.

If the item is not found, the report displays:

`PO item not found`

Why is EKPO required?

A PO can contain multiple items. The SES is associated with a particular PO item, so the report uses the SES item number to retrieve the correct item details.

## 6.4 Retrieve Vendor Details – LFA1

```
SELECT SINGLE *
  FROM lfa1
  INTO @gs_lfa1
  WHERE lifnr = @gs_ekko-lifnr.

IF sy-subrc = 0.
  gs_data-zsupplier_name = gs_lfa1-name1.
ENDIF.
```

### Explanation

- `LFA1` stores general vendor master information.
- `LIFNR` represents the vendor account number.
- The vendor number is retrieved from `GS_EKKO-LIFNR`.
- The matching vendor record is selected into `GS_LFA1`.
- `NAME1` contains the vendor name.

The vendor name is transferred to:

`GS_DATA-ZSUPPLIER_NAME`

This information is printed in the Pass Order.

The current code does not raise an error if the vendor record is not found.

## 6.5 Populate the Pass Order Structure

```
gs_data-zletter_date = sy-datum.

gs_data-zsubject = gs_ekpo-txz01.

gs_data-zloannumber = gs_ekko-zloanumber.
gs_data-zloa_date   = gs_ekko-kdatb.

gs_data-zcontract_date = gs_ekko-bedat.
gs_data-zcontract_from = gs_ekko-kdatb.
gs_data-zcontract_to = gs_ekko-kdate.

gs_data-zpo_no  = p_po_no.
gs_data-zses_no = p_ses_no.

gs_data-zservice_from = gs_essr-lzvon.
gs_data-zservice_to   = gs_essr-lzbis.

gs_data-zamount = gs_essr-lwert.
gs_data-zinvoice_amount = gs_essr-netwr.

gs_data-zletter_no = p_let_no.
```

This section transfers the retrieved information into the custom structure `GS_DATA`.

The structure acts as a common data source for the two Smart Forms.

### Field Mapping

| Output field      | Source field      | Description                 |
| ----------------- | ----------------- | --------------------------- |
| `ZLETTER_DATE`    | `SY-DATUM`        | Current system date         |
| `ZSUBJECT`        | `EKPO-TXZ01`      | PO item short text          |
| `ZLOANNUMBER`     | `EKKO-ZLOANUMBER` | LOA number                  |
| `ZLOA_DATE`       | `EKKO-KDATB`      | LOA date mapping            |
| `ZCONTRACT_DATE`  | `EKKO-BEDAT`      | PO document date            |
| `ZCONTRACT_FROM`  | `EKKO-KDATB`      | Contract start date mapping |
| `ZCONTRACT_TO`    | `EKKO-KDATE`      | Contract end date           |
| `ZPO_NO`          | `P_PO_NO`         | PO number                   |
| `ZSES_NO`         | `P_SES_NO`        | SES number                  |
| `ZSERVICE_FROM`   | `ESSR-LZVON`      | Service start date          |
| `ZSERVICE_TO`     | `ESSR-LZBIS`      | Service end date            |
| `ZAMOUNT`         | `ESSR-LWERT`      | SES value                   |
| `ZINVOICE_AMOUNT` | `ESSR-NETWR`      | Net value mapping           |
| `ZLETTER_NO`      | `P_LET_NO`        | Pass Order letter number    |

`SY-DATUM` is a system field that contains the current SAP system date.

Note: Verify the business meaning of the custom fields and the selected date/amount fields against your functional specification. In particular, confirm that `ESSR-NETWR` is the required invoice amount.

# 7. Invoice Details – EKBE and RBKP

The next section retrieves the invoice reference number and invoice date associated with the PO and SES.

```
CLEAR: gs_data-zinvoice_no,
       gs_data-zinvoice_date.

SELECT SINGLE
       rbkp~xblnr,
       rbkp~bldat
  FROM ekko
  INNER JOIN ekbe
    ON ekbe~ebeln = ekko~ebeln
  INNER JOIN rbkp
    ON rbkp~belnr = ekbe~belnr
   AND rbkp~lifnr = ekko~lifnr
  INTO (@gs_data-zinvoice_no,
        @gs_data-zinvoice_date)
  WHERE ekko~ebeln = @p_po_no
    AND ekbe~lfbnr = @p_ses_no
    AND ekbe~bewtp = 'Q'.

IF sy-subrc <> 0.
  gs_data-zinvoice_no   = gs_rbkp-xblnr.
  gs_data-zinvoice_date = gs_rbkp-bldat.
ENDIF.
```

## 7.1 Why are these tables used?

| Table | Purpose                                                     |
| ----- | ----------------------------------------------------------- |
| EKKO  | Provides the PO number and vendor                           |
| EKBE  | Provides PO history and invoice-related document references |
| RBKP  | Stores invoice document header information                  |

The required invoice fields are:

- `RBKP-XBLNR`: Invoice reference number.
- `RBKP-BLDAT`: Invoice document date.

## 7.2 JOIN Explanation

The report uses INNER JOIN to connect the purchasing document, PO history, and invoice header.

First JOIN: EKKO and EKBE

```
INNER JOIN ekbe
  ON ekbe~ebeln = ekko~ebeln
```

This connects the PO header with its purchasing history using the PO number.

Second JOIN: EKBE and RBKP

```
INNER JOIN rbkp
  ON rbkp~belnr = ekbe~belnr
 AND rbkp~lifnr = ekko~lifnr
```

This connects the invoice document number and vendor.

The invoice header details are retrieved based on the document references in the PO history.

## 7.3 WHERE Condition

```
WHERE ekko~ebeln = @p_po_no
  AND ekbe~lfbnr = @p_ses_no
  AND ekbe~bewtp = 'Q'.
```

- `EKKO-EBELN`: Restricts the selection to the entered PO.
- `EKBE-LFBNR`: Restricts the selection using the entered SES reference.
- `EKBE-BEWTP = 'Q'`: Restricts the PO history to the specified history category.

The selected invoice reference and date are directly stored in `GS_DATA`.

## 7.4 Important Code Observation

The current fallback uses:

```
gs_data-zinvoice_no   = gs_rbkp-xblnr.
gs_data-zinvoice_date = gs_rbkp-bldat.
```

However, `GS_RBKP` is not populated anywhere in the supplied code.

Therefore, if the invoice SELECT fails, the fallback does not retrieve another invoice. It copies initial values from the unpopulated work area.

A separate SELECT would be needed if a fallback invoice lookup is required.

Also, `SELECT SINGLE` should be reviewed if multiple invoices can be associated with the same PO and SES. A clear business rule is needed to determine which invoice should be selected.

# 8. Convert Invoice Amount into Words – SPELL_AMOUNT

```
DATA: lv_amount TYPE p DECIMALS 2,
      lv_words  TYPE spell.

CLEAR: lv_amount,
       lv_words,
       gs_data-zamount_words.

lv_amount = gs_data-zinvoice_amount.

CALL FUNCTION 'SPELL_AMOUNT'
  EXPORTING
    amount    = lv_amount
    currency  = 'INR'
    language  = 'E'
  IMPORTING
    in_words  = lv_words
  EXCEPTIONS
    not_found = 1
    too_large = 2
    OTHERS    = 3.

IF sy-subrc = 0.
  gs_data-zamount_words = lv_words-word.
ENDIF.
```

## 8.1 Purpose

This section converts the numeric invoice amount into English words.

For example, instead of displaying only a numeric amount, the Pass Order can display its equivalent amount in words.

## 8.2 Variable Explanation

| Variable                | Description                                       |
| ----------------------- | ------------------------------------------------- |
| `LV_AMOUNT`             | Stores the numeric amount with two decimal places |
| `LV_WORDS`              | Receives the converted amount in words            |
| `GS_DATA-ZAMOUNT_WORDS` | Stores the final amount-in-words text             |

## 8.3 Processing

Step 1 – Clear the variables

```
CLEAR: lv_amount,
       lv_words,
       gs_data-zamount_words.
```

Initializes the variables before conversion.

Step 2 – Assign the invoice amount

```
lv_amount = gs_data-zinvoice_amount.
```

The invoice amount is copied into `LV_AMOUNT`.

Step 3 – Call SPELL_AMOUNT

```
CALL FUNCTION 'SPELL_AMOUNT'
```

This standard SAP function module converts the numeric amount into words.

Exporting parameters:

- `AMOUNT`: Numeric amount.
- `CURRENCY`: `INR`, indicating Indian Rupees.
- `LANGUAGE`: `E`, indicating English.

Importing parameter:

\- `IN_WORDS`: Receives the converted amount in words.

Step 4 – Check the result

```
IF sy-subrc = 0.
  gs_data-zamount_words = lv_words-word.
ENDIF.
```

If the conversion is successful, the converted text is stored in `GS_DATA-ZAMOUNT_WORDS`.

This field is then available to the Smart Forms.

# 9. Retrieve Contract Amount – KONV

```
CLEAR gs_data-zcontract_amount.

SELECT SINGLE konv~kawrt
  FROM konv
  INTO @gs_data-zcontract_amount
  WHERE konv~knumv = @gs_ekko-knumv
    AND konv~kschl = 'JEXS'.

IF sy-subrc <> 0.
  CLEAR gs_data-zcontract_amount.
ENDIF.
```

## 9.1 Purpose

This section retrieves the required contract pricing amount from the pricing conditions associated with the PO.

## 9.2 KONV Table

`KONV` stores pricing condition information in classic SAP ERP.

Important fields used:

| Field   | Description                 |
| ------- | --------------------------- |
| `KNUMV` | Pricing document number     |
| `KSCHL` | Condition type              |
| `KAWRT` | Condition basis/value field |

## 9.3 Selection Logic

```
WHERE konv~knumv = @gs_ekko-knumv
  AND konv~kschl = 'JEXS'.
```

The report checks:

1. The pricing document number must match `EKKO-KNUMV`.
2. The condition type must be `JEXS`.

The selected `KAWRT` value is stored in:

`GS_DATA-ZCONTRACT_AMOUNT`

If no matching condition record exists, the field is cleared.

Important: Confirm that `JEXS` is the correct condition type for the required contract amount, including GST. Also verify whether multiple condition records can exist and whether further selection criteria are required. In S/4HANA, pricing data is generally stored in `PRCD_ELEMENTS`; check the system's data model before making changes.

# 10. CALL_SMARTFORMS – Generating the Pass Order

After all the required data has been retrieved, the report calls the two Smart Forms.

```
FORM call_smartforms.

  DATA: ls_control TYPE ssfctrlop,
        ls_output  TYPE ssfcrescl.
```

## 10.1 Declare Control and Output Structures

LS_CONTROL – SSFCTRLOP

Stores the control parameters required to execute a Smart Form.

LS_OUTPUT – SSFCRESCL

Receives the output information returned by the Smart Form execution.

# 11. First Smart Form – ZFS_NLC_PASS_ORDER

## 11.1 Retrieve Generated Function Module Name

```
CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING
    formname           = 'ZFS_NLC_PASS_ORDER'
  IMPORTING
    fm_name            = gv_fm_name1
  EXCEPTIONS
    no_form            = 1
    no_function_module = 2
    OTHERS             = 3.

IF sy-subrc <> 0.
  MESSAGE 'Smart Form 1 could not be generated'
          TYPE 'E'.
ENDIF.
```

### Explanation

A Smart Form is executed through a generated function module.

SAP generates this function module when the Smart Form is activated.

Instead of hardcoding the generated function module name, the report uses:

`SSF_FUNCTION_MODULE_NAME`

Exporting parameter:

\- `FORMNAME`: Name of the Smart Form, `ZFS_NLC_PASS_ORDER`.

Importing parameter:

\- `FM_NAME`: Returns the generated function module name.

The returned name is stored in `GV_FM_NAME1`.

If SAP cannot find the Smart Form or resolve its function module, the report displays an error message.

## 11.2 Set Control Parameters for Form 1

```
CLEAR ls_control.

ls_control-no_open   = space.
ls_control-no_close  = 'X'.
ls_control-preview   = 'X'.
ls_control-no_dialog = 'X'.
ls_control-device    = 'PRINTER'.
```

These parameters control how the first Smart Form is executed.

| Parameter   | Value   | Purpose                              |
| ----------- | ------- | ------------------------------------ |
| `NO_OPEN`   | Space   | Allows Form 1 to open the output job |
| `NO_CLOSE`  | X       | Keeps the output job open            |
| `PREVIEW`   | X       | Requests print preview               |
| `NO_DIALOG` | X       | Suppresses the print dialog          |
| `DEVICE`    | PRINTER | Uses printer output processing       |

### Why is NO_CLOSE set to X?

The report has two Smart Forms.

The first form should not close the output job because the second form must continue in the same output sequence.

Therefore, `NO_CLOSE = 'X'` keeps the job open after the first Smart Form.

## 11.3 Execute the First Smart Form

```
CLEAR ls_output.

CALL FUNCTION gv_fm_name1
  EXPORTING
    control_parameters = ls_control
    is_data            = gs_data
  IMPORTING
    job_output_info    = ls_output
  EXCEPTIONS
    formatting_error   = 1
    internal_error     = 2
    send_error         = 3
    user_canceled      = 4
    OTHERS             = 5.

IF sy-subrc <> 0.
  MESSAGE 'Error while executing Smart Form 1'
          TYPE 'E'.
ENDIF.
```

### Explanation

The generated function module of the first Smart Form is executed dynamically.

Exporting parameters

- `CONTROL_PARAMETERS`: Passes the control settings.
- `IS_DATA`: Passes the complete output structure `GS_DATA`.

Importing parameter

\- `JOB_OUTPUT_INFO`: Receives the output/job information.

The first Smart Form uses the data in `GS_DATA` to generate the first part of the Pass Order.

If an execution error occurs, the report displays:

`Error while executing Smart Form 1`

# 12. Second Smart Form – ZFS_NLC_PASS_ORDER2

The second Smart Form generates the final part of the Pass Order.

## 12.1 Retrieve Generated Function Module Name

```
CALL FUNCTION 'SSF_FUNCTION_MODULE_NAME'
  EXPORTING
    formname           = 'ZFS_NLC_PASS_ORDER2'
  IMPORTING
    fm_name            = gv_fm_name2
  EXCEPTIONS
    no_form            = 1
    no_function_module = 2
    OTHERS             = 3.

IF sy-subrc <> 0.
  MESSAGE 'Smart Form 2 could not be generated'
          TYPE 'E'.
ENDIF.
```

This retrieves the generated function module name for the second Smart Form.

The returned name is stored in `GV_FM_NAME2`.

## 12.2 Set Control Parameters for Form 2

```
CLEAR ls_control.

ls_control-no_open   = 'X'.
ls_control-no_close  = space.
ls_control-preview   = 'X'.
ls_control-no_dialog = 'X'.
ls_control-device    = 'PRINTER'.
```

The control parameters are different from those used for the first Smart Form.

| Parameter   | Value   | Purpose                        |
| ----------- | ------- | ------------------------------ |
| `NO_OPEN`   | X       | Does not open a new output job |
| `NO_CLOSE`  | Space   | Closes the existing output job |
| `PREVIEW`   | X       | Requests print preview         |
| `NO_DIALOG` | X       | Suppresses the print dialog    |
| `DEVICE`    | PRINTER | Uses printer output processing |

### Why are NO_OPEN and NO_CLOSE reversed?

The two Smart Forms are intended to be processed as one continuous output job.

First Smart Form:

- `NO_OPEN = Space`
- `NO_CLOSE = X`

The first form opens the output job but does not close it.

Second Smart Form:

- `NO_OPEN = X`
- `NO_CLOSE = Space`

The second form continues the existing output job and closes it after execution.

This is the important logic that allows both Smart Forms to be combined into one output sequence.

## 12.3 Execute the Second Smart Form

```
CLEAR ls_output.

CALL FUNCTION gv_fm_name2
  EXPORTING
    control_parameters = ls_control
    is_data            = gs_data
  IMPORTING
    job_output_info    = ls_output
  EXCEPTIONS
    formatting_error   = 1
    internal_error     = 2
    send_error         = 3
    user_canceled      = 4
    OTHERS             = 5.

IF sy-subrc <> 0.
  MESSAGE 'Error while executing Smart Form 2'
          TYPE 'E'.
ENDIF.
```

The second Smart Form is executed using the generated function module stored in `GV_FM_NAME2`.

The same `GS_DATA` structure is passed to the second form.

The second form adds its output to the existing Smart Form job and closes the job.

If execution fails, the report displays an error message.

# 13. Complete Process Flow

```
                  USER INPUT
                      |
          ---------------------------
          |            |            |
        PO No.       SES No.     Letter No.
          |            |            |
          ---------------------------
                      |
                      v
                START-OF-SELECTION
                      |
                      v
                   GET_DATA
                      |
          ---------------------------
          |            |            |
        EKKO         ESSR         EKPO
       PO Header     SES        PO Item
          |            |            |
          ---------------------------
                      |
                      v
                     LFA1
                  Vendor Name
                      |
                      v
               Populate GS_DATA
                      |
          ---------------------------
          |            |            |
       EKBE/RBKP    SPELL_AMOUNT   KONV
       Invoice      Amount Words   Contract
       Details                     Amount
          |            |            |
          ---------------------------
                      |
                      v
              CALL_SMARTFORMS
                      |
                      v
           SSF_FUNCTION_MODULE_NAME
                      |
                      v
            ZFS_NLC_PASS_ORDER
               First Smart Form
                      |
             Keep output job open
                      |
                      v
           SSF_FUNCTION_MODULE_NAME
                      |
                      v
            ZFS_NLC_PASS_ORDER2
               Second Smart Form
                      |
             Close output job
                      |
                      v
               PASS ORDER OUTPUT
```

# 14. Important Technical Observations

The following points should be reviewed when maintaining or enhancing the report.

| No. | Observation           | Explanation                                                                                           |
| --- | --------------------- | ----------------------------------------------------------------------------------------------------- |
| 1   | SES APPEND placement  | Move `APPEND` after the successful SELECT check.                                                      |
| 2   | GT_ESSR usage         | The internal table is not otherwise used in the supplied code.                                        |
| 3   | GS_RBKP fallback      | `GS_RBKP` is never populated, so the fallback does not retrieve invoice data.                         |
| 4   | GS_EKBE and GS_RSEG   | These work areas are declared but not used.                                                           |
| 5   | Invoice SELECT SINGLE | Check whether multiple invoices can exist for the same PO/SES and define the required selection rule. |
| 6   | Invoice amount        | Confirm that `ESSR-NETWR` is the correct amount for the Pass Order.                                   |
| 7   | Amount in words       | The report converts `ZINVOICE_AMOUNT`, so verify that this is the intended amount.                    |
| 8   | Contract amount       | Confirm that `JEXS` is the correct pricing condition for the required contract amount.                |
| 9   | Vendor validation     | The current code does not stop if the vendor is not found.                                            |
| 10  | Smart Form sequence   | Test that both forms appear in the correct order in one output job.                                   |

# 15. Testing Scenarios

| Test case                 | Input / condition                      | Expected result                                                      |
| ------------------------- | -------------------------------------- | -------------------------------------------------------------------- |
| Valid PO and SES          | Existing PO and related SES            | Pass Order data retrieved                                            |
| Invalid PO                | Non-existing PO number                 | Error: PO number does not exist                                      |
| SES belongs to another PO | Valid SES with a different PO          | Error: SES number does not belong to the entered PO                  |
| Invalid PO item           | SES item cannot be found in EKPO       | Error: PO item not found                                             |
| Vendor found              | Valid vendor number                    | Vendor name populated                                                |
| Invoice found             | Matching PO history and invoice record | Invoice reference and date populated                                 |
| Invoice not found         | No matching invoice history            | Invoice fields remain initial unless a valid fallback is implemented |
| Pricing condition found   | Matching `JEXS` record                 | Contract amount populated                                            |
| Pricing condition missing | No matching `JEXS` record              | Contract amount cleared                                              |
| Smart Form 1 failure      | Form cannot be resolved or executed    | Corresponding error message                                          |
| Smart Form 2 failure      | Form cannot be resolved or executed    | Corresponding error message                                          |
| Successful execution      | Both forms active and data available   | Combined Pass Order output                                           |

# 16. GitHub Project Summary

The `Z_NLC_PASS_ORDER` report automates the generation of NLC Pass Order documents using SAP ABAP and Smart Forms.

The report accepts a Purchase Order number, Service Entry Sheet number, and Pass Order letter number as input. It validates the PO and SES relationship, retrieves PO item and vendor information, and collects service-period, LOA, contract, and amount details.

It also retrieves invoice reference information through PO history and invoice header data, converts the invoice amount into words using the standard SAP function module `SPELL_AMOUNT`, and reads the selected contract pricing condition.

Finally, the report dynamically retrieves the generated function module names for two Smart Forms and executes them sequentially. The first Smart Form opens the output job without closing it, while the second Smart Form continues the same job and closes it.

This approach generates the complete Pass Order using two Smart Forms and a common custom output structure.

## 17. Key Concepts Learned

- Reading PO header information using `EKKO`.
- Retrieving PO item details using `EKPO`.
- Validating SES and PO relationships using `ESSR`.
- Retrieving vendor master information using `LFA1`.
- Understanding PO history and invoice document relationships using `EKBE` and `RBKP`.
- Using INNER JOIN to retrieve related data from multiple tables.
- Using `SELECT SINGLE` and checking `SY-SUBRC`.
- Mapping database fields into a custom ABAP structure.
- Using the standard function module `SPELL_AMOUNT`.
- Retrieving Smart Form generated function modules dynamically.
- Passing structured data to Smart Forms.
- Controlling Smart Form output jobs using `NO_OPEN` and `NO_CLOSE`.
- Executing multiple Smart Forms sequentially as one output job.

Project: NLC Pass Order Generation\
ABAP Report: `Z_NLC_PASS_ORDER`\
Output Structure: `ZST_NLC_PASS_ORDER`\
Smart Forms: `ZFS_NLC_PASS_ORDER`, `ZFS_NLC_PASS_ORDER2`

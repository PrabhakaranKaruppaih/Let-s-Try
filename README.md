Sub Do_Validations()

'Variables required for Validations
Dim ValidationBook As Workbook
Dim DataSheet As Worksheet  'Data
Dim CriteriaSheet As Worksheet  'Worksheet having the advanced criteria
Dim Filtered_DataSheet As Worksheet
Dim CurrencyCodes As Range
Dim CriteriaRange As Range
Dim ResultPage As Worksheet
Dim Difference As Double
'Validation Variable list ends

'For Period Validations
Dim RecordsOnValueDate As New ADODB.Recordset

'For Four Columns
Dim ConnectionForCodeCombinations As New ADODB.Connection
Dim CodeCombinationIDs As New ADODB.Recordset
'Four Columns Ends
xlapp.Visible = False
Set xlbook = xlapp.Workbooks.Open(fname, UpdateLinks:=False)
xlbook.Activate

Set ValidationBook = xlapp.Workbooks.Add    'Seperate Workbook for Validation Tasks
With ValidationBook
    .Parent.Visible = False 'Application
    'Set DataSheet = xlbook.Sheets(1).Copy(After:=ValidationBook.Sheets(ValidationBook.Sheets.Count))
    'Alternate way for the above code
    xlbook.Sheets(1).Copy After:=ValidationBook.Sheets(ValidationBook.Sheets.Count)
    Set DataSheet = ValidationBook.Sheets(ValidationBook.Sheets.Count)
        DataSheet.Name = "DataSheet"
    'alternate way ends
    For Each Sheet In ValidationBook.Sheets
        If Sheet.Name <> DataSheet.Name Then Sheet.Delete
    Next Sheet
    
    ConnectionForCodeCombinations.Open "DRIVER={Microsoft ODBC for Oracle};UID=apps;PWD=apps;SERVER=prod_dev;"
    
    'Period Validation Code begins
    For Each cell In DataSheet.UsedRange.Columns("F").Cells 'Value Date
        If cell.Row = 1 Then GoTo Next_Cell
        With RecordsOnValueDate
            '31-MAR-17 Sample Date Value for below code
            .Open "select count(1) from gl_period_statuses where application_id=101 and set_of_books_id=1 and closing_status='O'   and  to_date('" & cell.Value & "','DD/MM/YY') between start_date and end_date", ConnectionForCodeCombinations, 3, 1
                If .Fields(0).Value = 0 Then    'Blank records return count as zero
                    MsgBox "Period is not in open: " & Format(cell.Value, "DD-MMM-YY")
                    ValidationBook.Close savechanges:=False
                    Exit Sub
                End If
            .Close
        End With
Next_Cell:
    Next cell
    'Period Validation Code Ends
    
    Set CriteriaSheet = .Parent.Sheets.Add(After:=.Sheets(.Sheets.Count))
        CriteriaSheet.Name = "CriteriaSheet"
    Set Filtered_DataSheet = .Parent.Sheets.Add(After:=.Sheets(.Sheets.Count))
        Filtered_DataSheet.Name = "Filtered_Datasheet"
    Set ResultPage = .Parent.Sheets.Add(After:=.Sheets(.Sheets.Count))
        ResultPage.Name = "ResultPage"
    'Validation Procedure
    With DataSheet
        .Activate   'To Monitor
        Set CurrencyCodes = .UsedRange.Columns("L")
        'Preparing the Criteria for Advanced Filter
        With CriteriaSheet
            'Building the Criteria Range
            CurrencyCodes.Copy CriteriaSheet.Cells(1)
            CriteriaSheet.Activate 'To Monitor
            CriteriaSheet.Cells(1).CurrentRegion.RemoveDuplicates Columns:=1, Header:=xlYes 'Removing the Duplicates
            Set CriteriaRange = .UsedRange
            'Criteria Range Built

            For Each Row In CriteriaRange
                If Row.Row = 1 Then
                    GoTo Skip_Row
                End If
                
                'Filter on Criteriae one by one
                DataSheet.Activate 'To Monitor
                DataSheet.UsedRange.AdvancedFilter xlFilterCopy, CriteriaRange.Rows(2), Filtered_DataSheet.Cells(1) 'Filter based on the first 2 rows
                Filtered_DataSheet.Activate
                Filtered_DataSheet.Rows(1).Delete   'New Headers will generated for each Criteria
                
                'Sum of Credit - Sum of Debit
                ResultPage.Activate
                With ResultPage.Cells(1)
                    .FormulaArray = "=SUM(" & "Filtered_Datasheet!Q:Q" & ")-SUM(" & "Filtered_Datasheet!R:R" & ")"
                    .Value = Round(.Value, 2)
                End With
                If ResultPage.Cells(1).Value <> 0 Then
                    ValidationBook.Close savechanges:=False
                    MsgBox "Credit and Debit not Matching"
                    Exit Sub
                End If
                
                'Sum of Credit*CurrRates - Sum of Debit*CurrRates
                If Row.Cells(1).Value <> "USD" Then 'No currency rates for USD. At the same time Double check needed for others
                With ResultPage.Cells(1)
                    .FormulaArray = "=SUM(" & "Filtered_Datasheet!O:O*Filtered_Datasheet!Q:Q" & ")-SUM(" & "Filtered_Datasheet!O:O*Filtered_Datasheet!R:R" & ")"
                    .Value = Round(.Value, 2)
                End With
                End If
                
                If ResultPage.Cells(1).Value <> 0 Then
                    ValidationBook.Close savechanges:=False
                    MsgBox "Accounted Amount Not Tallies"
                    Exit Sub
                End If
                Row.Delete  'Deleting the processed criteria
                Filtered_DataSheet.Cells.ClearContents 'Sanitization
Skip_Row:
            Next Row
            
'Four Columns Code Begins
            .Activate 'To Monitor
            .Cells.ClearContents
            '(I,K,J,L)  'Column Sequence
            CriteriaSheet.Activate 'Next statement fills data on criteriasheet
            DataSheet.Columns("I:L").Copy CriteriaSheet.Columns(1) 'Gets Pasted on Criteria Sheet
            
            'Again Criteria Sheet
            .Activate
            .Columns(3).Cut
            .Columns(2).Insert Shift:=xlToRight 'Column Ordered as per requirements "J<->K"
            .UsedRange.RemoveDuplicates Columns:=Array(1, 2, 3, 4), Header:=xlYes   'Removing Duplicates
            .Rows(1).Delete 'Header No Longer Required
            'Sorting
            With .Sort.SortFields
                .Clear
                'Sort Fields
                For Each Column In CriteriaSheet.UsedRange.Columns  '4 Columns
                    .Add Key:=CriteriaSheet.UsedRange.Columns(Column.Column)   'Defaults:, SortOn:=xlSortOnValues, Order:=xlAscending, DataOption:=xlSortNormal
                Next Column
            End With
            
            With .Sort
                .SetRange CriteriaSheet.UsedRange
                .Header = xlNo
'                 Defaults
'                .MatchCase = False
'                .Orientation = xlTopToBottom
'                .SortMethod = xlPinYin
                .Apply
            End With
            
            'ConnectionForCodeCombinations.ConnectionString = "DRIVER={Microsoft ODBC for Oracle};UID=apps;PWD=apps;SERVER=prod_dev;"
            For Each Row In .UsedRange.Rows
                'Column Order Adjusted before sorted. So No need to supply in sequence suggested by Vishnu
                'If CodeCombinationIDs.State Then CodeCombinationIDs.Close
                'CodeCombinationIDs.Open , ConnectionForCodeCombinations, 3, 1
                With CodeCombinationIDs
                    'Oracle Code should not end with semicolon (;)
                    .Open "SELECT code_combination_id FROM gl_code_combinations_v a WHERE chart_of_accounts_id = 50153 AND template_id IS NULL AND segment1 = 'OIL' AND segment2 = " & Row.Cells(1).Value & " AND segment3 =  '" & Row.Cells(2).Value & "' AND segment4 =  " & Row.Cells(3).Value & " AND segment5 =  '" & Row.Cells(4).Value & "' AND enabled_flag = 'N'", ConnectionForCodeCombinations, 1, 3
                    If Not CodeCombinationIDs.BOF Then
                        MsgBox "The below combination is disabled in OFS. Please check with corporate" & Chr(13) & "OIL-" & Row.Cells(1).Value & "-" & Row.Cells(2).Value & "-" & Row.Cells(3).Value & "-" & Row.Cells(4).Value, vbCritical
                        ValidationBook.Close savechanges:=False 'The above code doesn't work if the book closes before
                        Exit Sub
                    End If
                    .Close  'Recordset
                End With
            Next Row
            ConnectionForCodeCombinations.Close 'Connection
'Four Columns Ends
        
        End With    'Criteria Sheet
    End With    'Data Sheet
    .Close savechanges:=False
End With
'Validation Procedure Ends

End Sub

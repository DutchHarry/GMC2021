# GMC2021

Basically this is a continuation of other work:
1. Get GMC data
2. Extract links from it to hearing and sanction papers
3. Extract text data from the PDFs that you get in this case

# Re 1.
# adapt the following to your needs
$line = "0000048"
$outputdir = "c:\temp\youroutdir\"
$useragent = "somerubbish"
$url = "https://www.gmc-uk.org/api/gmc/print/doctor?no=$line"
$a = invoke-webrequest $url -OutFile "$outputdir$line.PDF" -UserAgent "$useragent"

# Re 2.
$collectedlinksoutput = "c:\temp\gmclinks.txt"
$outputdir = "c:\temp\youroutdir\"
Get-ChildItem "$outputdir" | % {
  Add-Content $collectedoutput ( `
    (Get-Content $_) -match "^/URI \(http*" -notmatch '^*contact-us*' | Select -unique `
  )
}

# Re 3.
$iTextDllPath = "C:\Program Files\PackageManagement\NuGet\Packages\iTextSharp.5.5.13.2\lib\itextsharp.dll"
Add-Type -Path $iTextDllPath
$dllPath = "C:\Program Files\PackageManagement\NuGet\Packages\BouncyCastle.1.8.9\lib\BouncyCastle.Crypto.dll"
Add-Type -Path $dllPath
function Import-PDFText {
    <#
    .SYNOPSIS
        Import-PdfText
        Imports the raw text data of a PDF file as readable text.
    .DESCRIPTION
        Takes the path of a PDF file, loads the file in and converts the text into readable
        string data, before outputting it as a complete string.
    .EXAMPLE
        PS C:\> Import-PDFText -Path .\Test.pdf | Set-Content $env:Temp\test.txt

        Returns all of the text in Test.pdf as a string using StringBuilder and stores it to a
        file in the temp folder.
    .INPUTS
        Takes a file path as pipeline input.
    .OUTPUTS
        Outputs the entire text content of the PDF file as string.
    .NOTES
        Requires the iTextSharp library, which can be downloaded from here:
        https://sourceforge.net/projects/itextsharp/

        Place the DLL in the same folder as the script, and execute the function.
    #>
    [CmdletBinding()]
    Param(
        [Parameter(Mandatory, Position = 0, ValueFromPipeline)]
        [ValidateScript({ Test-Path $_ })]
        [string]
        $Path
    )
    begin {
        if (-not ([System.Management.Automation.PSTypeName]'iTextSharp.Text.Pdf.PdfReader').Type) {
            Add-Type -Path "$PSScriptRoot\itextsharp.dll"
        }
    }
    process {
        $Reader = New-Object 'iTextSharp.Text.Pdf.PdfReader' -ArgumentList $Path
        $PdfText = New-Object 'System.Text.StringBuilder'

        for ($Page = 1; $Page -le $Reader.NumberOfPages; $Page++) {
            $Strategy = New-Object 'iTextSharp.Text.Pdf.Parser.SimpleTextExtractionStrategy'
            $CurrentText = [iTextSharp.Text.Pdf.Parser.PdfTextExtractor]::GetTextFromPage($Reader, $Page, $Strategy)
            $PdfText.AppendLine([System.Text.Encoding]::UTF8.GetString([System.Text.ASCIIEncoding]::Convert([System.Text.Encoding]::Default, [System.Text.Encoding]::UTF8, [System.Text.Encoding]::Default.GetBytes($CurrentText))))
        }
        $Reader.Close()

        $PdfText.ToString()
    }
}

<#
# Usage:
$outputfile = "c:\temp\gmctextoutputs.txt"
$dirwithfiles = "c:\temp\youroutdir\"
$nl =[System.Environment]::Newline
Get-ChildItem $dirwithfiles | % {
  $data = $null
  $data = "$_" + $nl
  $pdf = $null
  $pdf = Import-PDFText -Path $_
  $data = $data + $pdf + $nl
  Add-Content $outputfile $data
}

#>

<#
After this you process the text outputs within SQL server
BULK INSERT as one lines with row delimiter '\n' and unpcik from there

#>

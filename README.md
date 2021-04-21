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

Getting the GMC numbers
Potentially there are about 5,500,000 valid GMC numbers
The tange from 0 to 5,000,000 exists of only 500k numbers with an additional checkdigit
Only the about 500,000 numbers with valid checkdigit were scanned.
The 5m and 6m ranges were in the past used for other puposes but contain some existing numbers.
Only 5,500,000 to 5,900,000 and 6,500,000 to 6,900,000 weren't scanned (yet).
Between 8,020,000 and 9,999,999 none were scanned either.

All other numbers were scanned.
But surely a few were missed (maybe temprarily offline?)

The result bearly 450k PDFs with GMC details

Transforming to text files
Tried a few with an Adobe Acrobat kit, with unsatisfactory results.
Tried with itextsharp (and poweshell) with better results
But finally went for processing with pdftotext (from xpdfreader)

# options -table and -raw aren't that good, so wh these options and within Poweshlell
Get-ChildItem "S:\_ue\gmc\R8\*.pdf" | % {
  $outputname = $_.Basename
  $output = "S:\_ue\gmc\R2\$outputname.txt"
  "G:\Portable\xpdf-tools-win-4.03\bin64\pdftotext.exe -nodiag -eol dos -nopgbrk -enc UTF-8 $_ $output " | CMD
}

# check if all went through:
$indir = "S:\_ue\gmc\whissues"
$outdir = "S:\_ue\gmc\txt"
Get-ChildItem $indir | % {
  $inname = $_.Basename;
  $outname = (Get-ChildItem "$outdir\$inname.*").Basename
  If ($inname -ne $outname) {
    Write-Host "$inname : MISSING"
  }
}


# Concatenation of TXT-files:
$delimiter = "££"
$indir =  "S:\_ue\gmc\R2\"
$destfile = "S:\_ue\gmc\gmcall.txt"
Get-ChildItem $indir -File -Filter *.txt | % {
	"$delimiter" | Out-File $destfile -Append; 
	$_.Name | Out-File $destfile -Append; 
	Get-Content $_.FullName | Out-File $destfile -Append
}


After aal this you can BULK INSERT into a SQL-server (or other) database and take the details apart from there.
That'll still need some handycrafting, as recognition of text even with pdftotext proved far from perfect.


# Re 2.
$collectedoutput = "S:\_ue\gmc\gmclinks.txt"
$indir = "S:\_ue\gmc\whissues"
Get-ChildItem "$indir\*.PDF" | % { `
  Add-Content $collectedoutput ( `
    (Get-Content "$_") -match "^/URI \(http*" -notmatch '^*contact-us*' | Select -unique `
  )
#  Write-Host "$_"
}


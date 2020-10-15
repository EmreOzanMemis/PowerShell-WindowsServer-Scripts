[CmdletBinding()]
Param
(
    [Parameter(Mandatory=$true, 
                ValueFromPipeline=$true,
                ValueFromPipelineByPropertyName=$true, 
                Position=0)]
    [ValidateNotNullOrEmpty()]
    [string[]]$ComputerName,


    [Parameter(Position=1)]
    [int]$ResultCount = 150
)

    $PipelineItems = @($input)
    if ($PipelineItems.Count)
    {
        $ComputerName = $PipelineItems
    }

$ComputerName | 
    Where-Object   { -not ($_ -as [ipaddress]) } |
    ForEach-Object {
        $null = Resolve-DnsName $_ -ErrorAction Stop
    }

$UseClearHostWhenRedrawing = $false
try {
    [System.Console]::SetCursorPosition(0, 0)
} catch [System.IO.IOException] {
    $UseClearHostWhenRedrawing = $true
}

Clear-Host

[array]$PingData = foreach($Computer in $ComputerName)
{
    @{
        'Name'       = $Computer
        'Pinger'     = New-Object -TypeName System.Net.NetworkInformation.Ping
        'Results'    = New-Object -TypeName System.Collections.Queue($ResultCount)
        'LastResult' = @{}
    }
}
foreach ($Item in $PingData)
{
    for ($Filler = 0; $Filler -lt $ResultCount; $Filler++)
    {
        $Item.Results.Enqueue('_')
    }
}

while ($true)
{

        [array]$PingTasks = foreach($Item in $PingData)
        {
            $Item.Pinger.SendPingAsync($Item.Name)

        }

        try {
            [Threading.Tasks.Task]::WaitAll($PingTasks)
        } catch [AggregateException] {
        }

        0..($PingTasks.Count-1) | ForEach-Object {
                
            $Task         = $PingTasks[$_]
            $ComputerData = $PingData[$_]

            if ($Task.Status -ne 'RanToCompletion')
            {
                $ComputerData.Results.Enqueue('?')
            }
            else
            {
                $ComputerData.LastResult = $Task.Result
                    
                switch ($Task.Result.Status)
                {
                    'Success'  { $ComputerData.Results.Enqueue('.') }
                    'TimedOut' { $ComputerData.Results.Enqueue('x') }
                }
                    
            }  
        }

        foreach ($Item in $PingData)
        {
            while ($Item.Results.Count -gt $ResultCount)
            {
                $null = $Item.Results.DeQueue()
            }
        }


        if ($UseClearHostWhenRedrawing)
        {
            Clear-Host
        }
        else
        {
            $CursorPosition = $Host.UI.RawUI.CursorPosition
            $CursorPosition.X = 0
            $CursorPosition.Y = 0
            $Host.UI.RawUI.CursorPosition = $CursorPosition
        }

        foreach ($Item in $PingData)
        {
            Write-Host (($Item.Results -join '') + ' | ') -NoNewline

            $PingText = if ($Item.LastResult.Status -eq 'Success')
            {
                if (1000 -le $Item.LastResult.RoundTripTime)
                {
                     '(999+ms)'
                }
                else
                {
                    '({0}ms)' -f $Item.LastResult.RoundTripTime.ToString().PadLeft(4, ' ')
                }
            }
            else
            {
                '(----ms)'
            }
            Write-Host "$PingText | " -NoNewline
            if ($Item.LastResult.Status -eq 'Success')
            {
                Write-Host ($Item.Name) -BackgroundColor DarkGreen
            }
            else
            {
                Write-Host ($Item.Name) -BackgroundColor DarkRed
            }
        }
        $Delay = 1000 - ($PingData.lastresult.roundtriptime | Sort-Object | Select-Object -Last 1)
        Start-Sleep -MilliSeconds $Delay
}

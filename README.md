# keyvaultcertupload

    trigger:
      branches:
        include:
          - main  
  # Trigger the pipeline on changes in the 'main branch

    pool:
      vmImage: 'windows-latest'  
  # Use the latest Windows image for the pipeline

    steps:
  # Step 1: Authenticate to Azure (Install Az module if not present)
    - task: AzureCLI@2
      inputs:
        azureSubscription: 'Azure Connection'  
  # The name of the Azure Service Connection (to be set in Azure DevOps)
      scriptType: 'pscore'
      scriptLocation: 'inlineScript'
      inlineScript: |
  # Install Az PowerShell module if not already installed
        if (-not (Get-Module -ListAvailable -Name Az)) {
            Install-Module -Name Az -AllowClobber -Force -Scope CurrentUser
        }
        
  # Define the required variables
      $clientId = " "  # Your Service Principal Client ID
    $tenantId = " "  # Your Azure Tenant ID
    $clientSecret = " "  # Fetch client secret from pipeline secret variables
    $secureClientSecret = ConvertTo-SecureString -String $clientSecret -AsPlainText -Force

# Create PSCredential object for authentication
    $credentials = New-Object System.Management.Automation.PSCredential ($clientId, $secureClientSecret)
        
  # Authenticate with Azure using Service Principal
    Connect-AzAccount -ServicePrincipal -TenantId $tenantId -Credential $credentials
  # Set the Azure subscription context
    Set-AzContext -SubscriptionId '786f8e51-ce3c-4d8e-9866-67444e6601d6'  
  # Your Subscription ID
    displayName: 'Authenticate to Azure'

  # Step 2: Download the Secure File (Certificate file) from Azure DevOps Secure Files
    - task: DownloadSecureFile@1
      inputs:
      secureFile: 'keyvaultvig.pfx'  # Name of the certificate file uploaded in Azure DevOps Secure Files

  # Step 3: Install Az PowerShell Module (if it's not already installed in the agent)
  - powershell: |
      Install-Module -Name Az -AllowClobber -Force -Scope CurrentUser
    displayName: 'Install Az PowerShell Module'

  # Step 4: Upload the Certificate to Azure Key Vault
    - powershell: |
  # Define the Key Vault name and certificate details
      $vaultName = "keyvaultvig"  # Your Key Vault name
      $certPath = "$(Agent.TempDirectory)\keyvaultvig.pfx"  # Path to the downloaded certificate
      $certName = "keyvaultvig"  # The name to assign to the certificate in Key Vault
      $certPassword = "key@123"  # Password for the certificate (use a secret variable for better security)

  # Convert the plain text password to a secure string
      $securePassword = ConvertTo-SecureString -String $certPassword -AsPlainText -Force

  # Check if the certificate already exists in Key Vault
      $existingCert = Get-AzKeyVaultCertificate -VaultName $vaultName -Name $certName -ErrorAction SilentlyContinue
      if ($existingCert) {
          Write-Host "Certificate already exists in Key Vault. Skipping upload."
      } else {
  # Import the certificate into Key Vault
          Import-AzKeyVaultCertificate `
            -VaultName $vaultName `
            -Name $certName `
            -FilePath $certPath `
            -Password $securePassword
          Write-Host "Certificate uploaded successfully to Key Vault."
      }
    displayName: 'Upload Certificate to Azure Key Vault'

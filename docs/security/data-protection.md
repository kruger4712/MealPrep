# Data Protection and Privacy Guide

## Overview
Comprehensive guide for data protection, privacy compliance, and security practices in the MealPrep application, ensuring user data safety and regulatory compliance including GDPR, CCPA, and other privacy regulations.

## Data Protection Principles

### Core Privacy Principles
1. **Lawfulness, Fairness, and Transparency**: Data processing must be lawful, fair, and transparent
2. **Purpose Limitation**: Data collected for specified, explicit, and legitimate purposes
3. **Data Minimization**: Adequate, relevant, and limited to what is necessary
4. **Accuracy**: Personal data must be accurate and kept up to date
5. **Storage Limitation**: Data kept for no longer than necessary
6. **Integrity and Confidentiality**: Appropriate security measures for data protection
7. **Accountability**: Controller must demonstrate compliance

---

## Data Classification

### Personal Data Categories

#### **Basic Personal Information**
- **Data Types**: Name, email address, profile photos
- **Classification**: Personal Data (PII)
- **Retention**: Account lifetime + 30 days after deletion
- **Access Level**: User, Admin
- **Encryption**: At rest and in transit

#### **Family Composition Data**
- **Data Types**: Family member names, ages, relationships
- **Classification**: Personal Data (PII)
- **Retention**: Account lifetime + 30 days after deletion
- **Access Level**: User only
- **Encryption**: At rest and in transit

#### **Dietary and Health Information**
- **Data Types**: Allergies, dietary restrictions, health goals
- **Classification**: Sensitive Personal Data (Special Category)
- **Retention**: Account lifetime + 90 days after deletion
- **Access Level**: User only
- **Encryption**: Enhanced encryption (AES-256)

#### **Behavioral Data**
- **Data Types**: Recipe preferences, cooking history, app usage
- **Classification**: Personal Data (Behavioral)
- **Retention**: 24 months after last activity
- **Access Level**: User, Aggregated Analytics
- **Encryption**: At rest and in transit

#### **AI Interaction Data**
- **Data Types**: AI requests, suggestions accepted/rejected
- **Classification**: Personal Data (AI Training)
- **Retention**: 12 months for personalization, anonymized for model improvement
- **Access Level**: User, AI Service (anonymized)
- **Encryption**: At rest and in transit

### Data Classification Matrix

| Data Type | Classification | Sensitivity | Retention Period | Encryption Level |
|-----------|---------------|-------------|------------------|------------------|
| Email/Password | PII | High | Account + 30d | AES-256 |
| Name/Profile | PII | Medium | Account + 30d | AES-256 |
| Family Data | PII | High | Account + 30d | AES-256 |
| Health Data | Special Category | Critical | Account + 90d | AES-256 + Field Level |
| Preferences | Behavioral | Medium | 24 months | AES-256 |
| Usage Analytics | Aggregated | Low | 36 months | AES-256 |
| AI Interactions | Behavioral | Medium | 12 months | AES-256 |

---

## GDPR Compliance Implementation

### Legal Basis for Processing

#### **Consent (Article 6(1)(a))**
- **Use Cases**: Marketing communications, optional features
- **Implementation**: Explicit opt-in checkboxes, granular consent management
- **Withdrawal**: Easy withdrawal mechanism in user settings

```csharp
public class ConsentManager
{
    public async Task<bool> HasValidConsentAsync(int userId, ConsentType consentType)
    {
        var consent = await _context.UserConsents
            .Where(c => c.UserId == userId && c.ConsentType == consentType)
            .OrderByDescending(c => c.CreatedDate)
            .FirstOrDefaultAsync();

        return consent?.IsGranted == true && 
               consent.ExpiryDate > DateTime.UtcNow;
    }

    public async Task RecordConsentAsync(int userId, ConsentType consentType, bool isGranted)
    {
        var consent = new UserConsent
        {
            UserId = userId,
            ConsentType = consentType,
            IsGranted = isGranted,
            CreatedDate = DateTime.UtcNow,
            ExpiryDate = isGranted ? DateTime.UtcNow.AddYears(2) : null,
            IpAddress = _httpContextAccessor.HttpContext?.Connection?.RemoteIpAddress?.ToString(),
            UserAgent = _httpContextAccessor.HttpContext?.Request?.Headers["User-Agent"].ToString()
        };

        _context.UserConsents.Add(consent);
        await _context.SaveChangesAsync();
    }
}
```

#### **Contract Performance (Article 6(1)(b))**
- **Use Cases**: Core meal planning services, account management
- **Implementation**: Automatic processing for service delivery
- **Scope**: Limited to data necessary for service provision

#### **Legitimate Interest (Article 6(1)(f))**
- **Use Cases**: Service improvement, fraud prevention, security
- **Implementation**: Legitimate Interest Assessment (LIA) documentation
- **Balancing Test**: Regular review of data subject rights vs. business needs

### Data Subject Rights Implementation

#### **Right of Access (Article 15)**
```csharp
[HttpGet("data-export")]
[Authorize]
public async Task<IActionResult> ExportUserData()
{
    var userId = GetCurrentUserId();
    var userData = await _dataExportService.GenerateUserDataExportAsync(userId);
    
    var fileName = $"mealprep-data-export-{userId}-{DateTime.UtcNow:yyyy-MM-dd}.json";
    var jsonData = JsonSerializer.Serialize(userData, new JsonSerializerOptions 
    { 
        WriteIndented = true 
    });
    
    return File(Encoding.UTF8.GetBytes(jsonData), "application/json", fileName);
}

public class UserDataExport
{
    public UserProfile Profile { get; set; }
    public List<FamilyMemberData> FamilyMembers { get; set; }
    public List<RecipeData> Recipes { get; set; }
    public List<MenuPlanData> MenuPlans { get; set; }
    public List<AiInteractionData> AiInteractions { get; set; }
    public ConsentHistory ConsentHistory { get; set; }
    public DataProcessingLog ProcessingLog { get; set; }
    public DateTime ExportedAt { get; set; }
    public string ExportVersion { get; set; }
}
```

#### **Right to Rectification (Article 16)**
```csharp
[HttpPut("profile")]
[Authorize]
public async Task<IActionResult> UpdateProfile([FromBody] UpdateProfileRequest request)
{
    var userId = GetCurrentUserId();
    
    // Validate and sanitize input
    var validationResult = await _validator.ValidateAsync(request);
    if (!validationResult.IsValid)
        return BadRequest(validationResult.Errors);

    // Update user profile
    await _userService.UpdateProfileAsync(userId, request);
    
    // Log data modification for audit trail
    await _auditService.LogDataModificationAsync(userId, "Profile", "Update", request);
    
    return Ok();
}
```

#### **Right to Erasure (Article 17)**
```csharp
[HttpDelete("account")]
[Authorize]
public async Task<IActionResult> DeleteAccount([FromBody] AccountDeletionRequest request)
{
    var userId = GetCurrentUserId();
    
    // Validate deletion request
    if (!await _authService.ValidatePasswordAsync(userId, request.Password))
        return Unauthorized("Invalid password");

    // Initiate data deletion process
    await _dataDeletionService.InitiateAccountDeletionAsync(userId, request.Reason);
    
    return Ok(new { Message = "Account deletion initiated. You will receive confirmation via email." });
}

public class DataDeletionService
{
    public async Task InitiateAccountDeletionAsync(int userId, string reason)
    {
        // Create deletion request with grace period
        var deletionRequest = new AccountDeletionRequest
        {
            UserId = userId,
            RequestedAt = DateTime.UtcNow,
            ScheduledDeletionDate = DateTime.UtcNow.AddDays(30), // 30-day grace period
            Reason = reason,
            Status = DeletionStatus.Pending
        };

        _context.AccountDeletionRequests.Add(deletionRequest);
        await _context.SaveChangesAsync();

        // Send confirmation email with cancellation link
        await _emailService.SendAccountDeletionConfirmationAsync(userId, deletionRequest.Id);
    }

    public async Task ExecuteScheduledDeletionsAsync()
    {
        var pendingDeletions = await _context.AccountDeletionRequests
            .Where(r => r.Status == DeletionStatus.Pending && 
                       r.ScheduledDeletionDate <= DateTime.UtcNow)
            .ToListAsync();

        foreach (var deletion in pendingDeletions)
        {
            await PerformDataDeletionAsync(deletion.UserId);
            deletion.Status = DeletionStatus.Completed;
            deletion.CompletedAt = DateTime.UtcNow;
        }

        await _context.SaveChangesAsync();
    }

    private async Task PerformDataDeletionAsync(int userId)
    {
        // Delete user data in specific order to maintain referential integrity
        await _context.AiSuggestionRequests.Where(r => r.UserId == userId).ExecuteDeleteAsync();
        await _context.MenuPlans.Where(m => m.UserId == userId).ExecuteDeleteAsync();
        await _context.Recipes.Where(r => r.CreatedByUserId == userId).ExecuteDeleteAsync();
        await _context.FamilyMembers.Where(f => f.UserId == userId).ExecuteDeleteAsync();
        await _context.Users.Where(u => u.Id == userId).ExecuteDeleteAsync();

        // Log deletion for compliance audit
        await _auditService.LogDataDeletionAsync(userId, "Complete account deletion");
    }
}
```

#### **Right to Data Portability (Article 20)**
```csharp
[HttpGet("data-export/portable")]
[Authorize]
public async Task<IActionResult> ExportPortableData()
{
    var userId = GetCurrentUserId();
    var portableData = await _dataExportService.GeneratePortableDataExportAsync(userId);
    
    // Create structured, machine-readable format
    var export = new PortableDataExport
    {
        User = portableData.Profile,
        Recipes = portableData.Recipes.Select(r => new PortableRecipe
        {
            Name = r.Name,
            Description = r.Description,
            Instructions = r.Instructions,
            PrepTime = r.PrepTimeMinutes,
            CookTime = r.CookTimeMinutes,
            Ingredients = r.Ingredients,
            Tags = r.Tags
        }).ToList(),
        Preferences = portableData.FamilyMembers.SelectMany(fm => fm.Preferences).ToList(),
        ExportFormat = "JSON-LD",
        ExportedAt = DateTime.UtcNow
    };
    
    return Ok(export);
}
```

---

## Data Security Implementation

### Encryption at Rest

#### **Database Encryption**
```csharp
// Field-level encryption for sensitive data
public class EncryptedHealthGoals
{
    private readonly IDataProtectionProvider _dataProtectionProvider;
    private readonly IDataProtector _protector;

    public EncryptedHealthGoals(IDataProtectionProvider dataProtectionProvider)
    {
        _dataProtectionProvider = dataProtectionProvider;
        _protector = _dataProtectionProvider.CreateProtector("HealthGoals.v1");
    }

    public string Encrypt(string plainText)
    {
        if (string.IsNullOrEmpty(plainText))
            return plainText;

        return _protector.Protect(plainText);
    }

    public string Decrypt(string cipherText)
    {
        if (string.IsNullOrEmpty(cipherText))
            return cipherText;

        try
        {
            return _protector.Unprotect(cipherText);
        }
        catch (CryptographicException)
        {
            // Handle decryption failures gracefully
            return "[Encrypted Data - Unable to Decrypt]";
        }
    }
}

// Entity configuration for encrypted fields
public class FamilyMemberConfiguration : IEntityTypeConfiguration<FamilyMember>
{
    public void Configure(EntityTypeBuilder<FamilyMember> builder)
    {
        builder.Property(e => e.HealthGoals)
            .HasConversion(
                v => EncryptionHelper.Encrypt(v),
                v => EncryptionHelper.Decrypt(v)
            );
    }
}
```

#### **File Storage Encryption**
```csharp
public class SecureFileStorage
{
    private readonly BlobServiceClient _blobServiceClient;
    private readonly IDataProtectionProvider _dataProtectionProvider;

    public async Task<string> UploadEncryptedFileAsync(Stream fileStream, string fileName)
    {
        var protector = _dataProtectionProvider.CreateProtector($"FileStorage.{fileName}");
        
        using var encryptedStream = new MemoryStream();
        using var cryptoStream = new ProtectedStream(encryptedStream, protector);
        
        await fileStream.CopyToAsync(cryptoStream);
        encryptedStream.Position = 0;

        var blobName = $"{Guid.NewGuid()}/{fileName}";
        var blobClient = _blobServiceClient
            .GetBlobContainerClient("secure-files")
            .GetBlobClient(blobName);

        await blobClient.UploadAsync(encryptedStream, new BlobUploadOptions
        {
            HttpHeaders = new BlobHttpHeaders
            {
                ContentType = "application/octet-stream"
            },
            Metadata = new Dictionary<string, string>
            {
                ["OriginalFileName"] = fileName,
                ["EncryptedAt"] = DateTime.UtcNow.ToString("O"),
                ["EncryptionVersion"] = "v1"
            }
        });

        return blobName;
    }
}
```

### Encryption in Transit

#### **HTTPS Configuration**
```csharp
// Program.cs - HTTPS enforcement
builder.Services.AddHttpsRedirection(options =>
{
    options.RedirectStatusCode = StatusCodes.Status307TemporaryRedirect;
    options.HttpsPort = 443;
});

builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
    app.UseHttpsRedirection();
}
```

#### **API Security Headers**
```csharp
public class SecurityHeadersMiddleware
{
    public async Task InvokeAsync(HttpContext context, RequestDelegate next)
    {
        // Security headers for data protection
        context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
        context.Response.Headers.Add("X-Frame-Options", "DENY");
        context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
        context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
        context.Response.Headers.Add("Content-Security-Policy", 
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline';");

        await next(context);
    }
}
```

---

## Privacy by Design Implementation

### Data Minimization

#### **Selective Data Collection**
```csharp
public class DataMinimizationService
{
    public async Task<AiSuggestionRequest> CreateMinimalAiRequestAsync(int userId, MealPlanningRequest request)
    {
        // Only collect data necessary for AI processing
        var familyData = await _context.FamilyMembers
            .Where(fm => fm.UserId == userId && fm.IsActive)
            .Select(fm => new MinimalFamilyData
            {
                // Only essential data for AI
                Age = fm.Age,
                SpiceToleranceLevel = fm.SpiceToleranceLevel,
                CookingSkillLevel = fm.CookingSkillLevel,
                CriticalAllergies = fm.Preferences
                    .Where(p => p.PreferenceType == "Allergy" && p.Severity == "Critical")
                    .Select(p => p.PreferenceValue)
                    .ToList()
                // Exclude: Name, detailed health goals, non-critical preferences
            })
            .ToListAsync();

        return new AiSuggestionRequest
        {
            UserId = userId,
            FamilyData = familyData,
            RequestedAt = DateTime.UtcNow,
            // Auto-delete after processing
            DeleteAfter = DateTime.UtcNow.AddDays(7)
        };
    }
}
```

#### **Purpose-Specific Data Views**
```csharp
public class PrivacyAwareDataService
{
    public async Task<UserProfileDto> GetUserProfileForPurposeAsync(int userId, DataPurpose purpose)
    {
        var baseQuery = _context.Users.Where(u => u.Id == userId);

        return purpose switch
        {
            DataPurpose.Authentication => await baseQuery
                .Select(u => new UserProfileDto
                {
                    Id = u.Id,
                    Email = u.Email,
                    IsActive = u.IsActive
                    // Minimal data for auth
                })
                .FirstOrDefaultAsync(),

            DataPurpose.MealPlanning => await baseQuery
                .Select(u => new UserProfileDto
                {
                    Id = u.Id,
                    FirstName = u.FirstName,
                    CookingPreferences = u.FamilyMembers
                        .Where(fm => fm.IsActive)
                        .Select(fm => new CookingPreference
                        {
                            SkillLevel = fm.CookingSkillLevel,
                            AvailableTime = fm.PreferredCookingTime
                        })
                        .ToList()
                    // No personal details beyond cooking needs
                })
                .FirstOrDefaultAsync(),

            DataPurpose.Marketing => await baseQuery
                .Where(u => u.MarketingConsent == true)
                .Select(u => new UserProfileDto
                {
                    Email = u.Email,
                    FirstName = u.FirstName,
                    PreferredCuisines = u.FamilyMembers
                        .SelectMany(fm => fm.Preferences)
                        .Where(p => p.PreferenceType == "Cuisine" && 
                               p.MarketingConsentForPreference == true)
                        .Select(p => p.PreferenceValue)
                        .ToList()
                    // Only with explicit marketing consent
                })
                .FirstOrDefaultAsync(),

            _ => throw new ArgumentException($"Unsupported data purpose: {purpose}")
        };
    }
}
```

### Automated Data Lifecycle Management

#### **Data Retention Automation**
```csharp
public class DataRetentionService : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await PerformDataRetentionTasksAsync();
            await Task.Delay(TimeSpan.FromHours(24), stoppingToken); // Daily cleanup
        }
    }

    private async Task PerformDataRetentionTasksAsync()
    {
        var cutoffDates = GetRetentionCutoffDates();

        // Clean up AI request data (12 months)
        await _context.AiSuggestionRequests
            .Where(r => r.CreatedDate < cutoffDates.AiRequests)
            .ExecuteDeleteAsync();

        // Clean up behavioral data (24 months)
        await _context.UserActivityLogs
            .Where(l => l.CreatedDate < cutoffDates.BehavioralData)
            .ExecuteDeleteAsync();

        // Clean up expired consent records (3 years)
        await _context.UserConsents
            .Where(c => c.ExpiryDate < cutoffDates.ConsentRecords)
            .ExecuteDeleteAsync();

        // Anonymize old recipe data (beyond personal retention)
        await AnonymizeOldRecipeDataAsync(cutoffDates.PersonalData);

        _logger.LogInformation("Data retention cleanup completed");
    }

    private async Task AnonymizeOldRecipeDataAsync(DateTime cutoff)
    {
        var oldRecipes = await _context.Recipes
            .Where(r => r.CreatedDate < cutoff && r.CreatedByUserId.HasValue)
            .ToListAsync();

        foreach (var recipe in oldRecipes)
        {
            // Remove personal association but keep recipe for community
            recipe.CreatedByUserId = null;
            recipe.OriginalCreatorId = recipe.CreatedByUserId; // For analytics only
            recipe.AnonymizedAt = DateTime.UtcNow;
        }

        await _context.SaveChangesAsync();
    }
}
```

---

## Consent Management

### Granular Consent System

#### **Consent Types and Management**
```csharp
public enum ConsentType
{
    ServiceTerms,           // Required for service
    PrivacyPolicy,          // Required for service
    MarketingEmails,        // Optional
    PersonalizedAds,        // Optional
    DataAnalytics,          // Optional
    ThirdPartySharing,      // Optional
    AiModelImprovement,     // Optional
    CommunityFeatures      // Optional
}

public class ConsentManagementService
{
    public async Task<ConsentStatus> GetConsentStatusAsync(int userId)
    {
        var consents = await _context.UserConsents
            .Where(c => c.UserId == userId)
            .GroupBy(c => c.ConsentType)
            .Select(g => new ConsentRecord
            {
                ConsentType = g.Key,
                CurrentConsent = g.OrderByDescending(c => c.CreatedDate).First()
            })
            .ToListAsync();

        return new ConsentStatus
        {
            UserId = userId,
            Consents = consents.ToDictionary(c => c.ConsentType, c => c.CurrentConsent),
            LastUpdated = consents.Max(c => c.CurrentConsent.CreatedDate),
            RequiresUpdate = HasExpiredConsents(consents) || HasMissingRequiredConsents(consents)
        };
    }

    public async Task UpdateConsentAsync(int userId, ConsentUpdateRequest request)
    {
        foreach (var consentUpdate in request.Consents)
        {
            var consent = new UserConsent
            {
                UserId = userId,
                ConsentType = consentUpdate.ConsentType,
                IsGranted = consentUpdate.IsGranted,
                CreatedDate = DateTime.UtcNow,
                ExpiryDate = CalculateExpiryDate(consentUpdate.ConsentType, consentUpdate.IsGranted),
                IpAddress = request.IpAddress,
                UserAgent = request.UserAgent,
                ConsentMethod = request.ConsentMethod // Web form, API, etc.
            };

            _context.UserConsents.Add(consent);

            // Trigger consent-specific actions
            await HandleConsentChangeAsync(userId, consentUpdate);
        }

        await _context.SaveChangesAsync();
    }

    private async Task HandleConsentChangeAsync(int userId, ConsentUpdate consentUpdate)
    {
        switch (consentUpdate.ConsentType)
        {
            case ConsentType.MarketingEmails when !consentUpdate.IsGranted:
                await _emailService.UnsubscribeFromMarketingAsync(userId);
                break;

            case ConsentType.DataAnalytics when !consentUpdate.IsGranted:
                await _analyticsService.OptOutUserAsync(userId);
                break;

            case ConsentType.AiModelImprovement when !consentUpdate.IsGranted:
                await _aiService.ExcludeFromModelTrainingAsync(userId);
                break;
        }
    }
}
```

### Cookie Consent Management

#### **Frontend Cookie Consent**
```typescript
// Cookie consent management
export interface CookieConsent {
  necessary: boolean;
  analytics: boolean;
  marketing: boolean;
  preferences: boolean;
}

export class CookieConsentManager {
  private static readonly CONSENT_KEY = 'mealprep_cookie_consent';
  private static readonly CONSENT_VERSION = '1.0';

  static getConsent(): CookieConsent | null {
    const stored = localStorage.getItem(this.CONSENT_KEY);
    if (!stored) return null;

    try {
      const parsed = JSON.parse(stored);
      if (parsed.version !== this.CONSENT_VERSION) {
        // Version mismatch, require new consent
        this.clearConsent();
        return null;
      }
      return parsed.consent;
    } catch {
      return null;
    }
  }

  static saveConsent(consent: CookieConsent): void {
    const consentData = {
      consent,
      version: this.CONSENT_VERSION,
      timestamp: new Date().toISOString()
    };

    localStorage.setItem(this.CONSENT_KEY, JSON.stringify(consentData));
    this.applyConsent(consent);
  }

  private static applyConsent(consent: CookieConsent): void {
    // Apply analytics consent
    if (consent.analytics && window.gtag) {
      window.gtag('consent', 'update', {
        analytics_storage: 'granted'
      });
    }

    // Apply marketing consent
    if (consent.marketing && window.gtag) {
      window.gtag('consent', 'update', {
        ad_storage: 'granted',
        ad_user_data: 'granted',
        ad_personalization: 'granted'
      });
    }

    // Remove non-consented cookies
    if (!consent.analytics) {
      this.removeCookiesByCategory('analytics');
    }

    if (!consent.marketing) {
      this.removeCookiesByCategory('marketing');
    }
  }

  private static removeCookiesByCategory(category: string): void {
    const cookiesToRemove = this.getCookiesByCategory(category);
    cookiesToRemove.forEach(cookieName => {
      document.cookie = `${cookieName}=; expires=Thu, 01 Jan 1970 00:00:00 GMT; path=/`;
    });
  }
}
```

---

## Data Breach Response

### Incident Response Plan

#### **Breach Detection and Classification**
```csharp
public class DataBreachDetectionService
{
    public async Task<BreachAssessment> AssessSecurityIncidentAsync(SecurityIncident incident)
    {
        var assessment = new BreachAssessment
        {
            IncidentId = incident.Id,
            Severity = DetermineSeverity(incident),
            AffectedDataTypes = IdentifyAffectedDataTypes(incident),
            AffectedUserCount = await EstimateAffectedUsersAsync(incident),
            GeographicScope = DetermineGeographicScope(incident),
            RegulatoryNotificationRequired = false
        };

        // Determine if regulatory notification is required
        assessment.RegulatoryNotificationRequired = 
            assessment.Severity >= BreachSeverity.High ||
            assessment.AffectedDataTypes.Contains(DataType.HealthInformation) ||
            assessment.AffectedUserCount > 250; // GDPR threshold

        // Auto-initiate response procedures
        if (assessment.RegulatoryNotificationRequired)
        {
            await InitiateBreachResponseAsync(assessment);
        }

        return assessment;
    }

    private async Task InitiateBreachResponseAsync(BreachAssessment assessment)
    {
        // Step 1: Immediate containment
        await _securityService.ContainBreachAsync(assessment.IncidentId);

        // Step 2: Preserve evidence
        await _auditService.PreserveIncidentEvidenceAsync(assessment.IncidentId);

        // Step 3: Notify breach response team
        await _notificationService.NotifyBreachResponseTeamAsync(assessment);

        // Step 4: Start regulatory notification timer (72 hours for GDPR)
        await _taskScheduler.ScheduleRegulatoryNotificationAsync(assessment, TimeSpan.FromHours(72));

        // Step 5: Prepare user notification if required
        if (assessment.RequiresUserNotification)
        {
            await PrepareUserNotificationAsync(assessment);
        }
    }
}
```

#### **Breach Notification Templates**
```csharp
public class BreachNotificationService
{
    public async Task SendUserBreachNotificationAsync(int userId, BreachNotification notification)
    {
        var user = await _userService.GetUserAsync(userId);
        
        var email = new EmailMessage
        {
            To = user.Email,
            Subject = "Important Security Notice - MealPrep Account",
            Template = "BreachNotification",
            Variables = new Dictionary<string, object>
            {
                ["UserName"] = user.FirstName,
                ["IncidentDate"] = notification.IncidentDate.ToString("MMMM dd, yyyy"),
                ["DataAffected"] = notification.AffectedDataTypes,
                ["ActionsTaken"] = notification.RemediationActions,
                ["RecommendedActions"] = notification.UserActions,
                ["ContactInfo"] = notification.ContactInformation
            }
        };

        await _emailService.SendEmailAsync(email);

        // Log notification for compliance
        await _auditService.LogBreachNotificationAsync(userId, notification.IncidentId);
    }

    public async Task SubmitRegulatoryNotificationAsync(BreachAssessment assessment)
    {
        var gdprNotification = new GDPRBreachNotification
        {
            IncidentReference = assessment.IncidentId,
            DataController = GetDataControllerInformation(),
            IncidentDescription = assessment.Description,
            DataCategories = assessment.AffectedDataTypes,
            DataSubjectCategories = assessment.AffectedUserTypes,
            ApproximateNumberOfDataSubjects = assessment.AffectedUserCount,
            ConsequencesOfBreach = assessment.PotentialConsequences,
            MeasuresTaken = assessment.RemediationActions,
            ContactDetails = GetDataProtectionOfficerContact()
        };

        await _regulatoryService.SubmitGDPRNotificationAsync(gdprNotification);
    }
}
```

---

## Monitoring and Compliance

### Privacy Compliance Monitoring

#### **Automated Compliance Checks**
```csharp
public class PrivacyComplianceMonitor : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await PerformComplianceChecksAsync();
            await Task.Delay(TimeSpan.FromHours(6), stoppingToken); // Check every 6 hours
        }
    }

    private async Task PerformComplianceChecksAsync()
    {
        var checks = new List<ComplianceCheck>
        {
            await CheckDataRetentionComplianceAsync(),
            await CheckConsentValidityAsync(),
            await CheckDataProcessingLegalBasisAsync(),
            await CheckEncryptionComplianceAsync(),
            await CheckAccessLogIntegrityAsync()
        };

        var failedChecks = checks.Where(c => !c.Passed).ToList();
        
        if (failedChecks.Any())
        {
            await _alertService.SendComplianceAlertAsync(failedChecks);
        }

        await _auditService.LogComplianceCheckAsync(checks);
    }

    private async Task<ComplianceCheck> CheckDataRetentionComplianceAsync()
    {
        var overRetentionData = await _context.Users
            .Where(u => !u.IsActive && 
                       u.DeactivatedAt < DateTime.UtcNow.AddDays(-30))
            .CountAsync();

        return new ComplianceCheck
        {
            CheckName = "Data Retention",
            Passed = overRetentionData == 0,
            Description = overRetentionData > 0 
                ? $"{overRetentionData} user records exceed retention period"
                : "All data within retention limits",
            Severity = overRetentionData > 0 ? ComplianceSeverity.High : ComplianceSeverity.Low
        };
    }
}
```

### Data Processing Records

#### **Article 30 GDPR Records**
```csharp
public class DataProcessingRecord
{
    public string ProcessingActivity { get; set; }
    public string LegalBasis { get; set; }
    public string[] DataCategories { get; set; }
    public string[] DataSubjectCategories { get; set; }
    public string Purpose { get; set; }
    public string[] Recipients { get; set; }
    public string RetentionPeriod { get; set; }
    public string SecurityMeasures { get; set; }
    public DateTime LastReviewed { get; set; }
}

public static class ProcessingActivities
{
    public static readonly DataProcessingRecord UserAccountManagement = new()
    {
        ProcessingActivity = "User Account Management",
        LegalBasis = "Contract performance (GDPR Art. 6(1)(b))",
        DataCategories = new[] { "Contact data", "Account credentials", "Profile information" },
        DataSubjectCategories = new[] { "App users", "Prospects" },
        Purpose = "Provision of meal planning services",
        Recipients = new[] { "Internal team", "Cloud hosting provider" },
        RetentionPeriod = "Account lifetime + 30 days",
        SecurityMeasures = "Encryption at rest and in transit, access controls, audit logging",
        LastReviewed = DateTime.Parse("2024-01-15")
    };

    public static readonly DataProcessingRecord AiPersonalization = new()
    {
        ProcessingActivity = "AI-Powered Meal Suggestions",
        LegalBasis = "Legitimate interest (GDPR Art. 6(1)(f))",
        DataCategories = new[] { "Dietary preferences", "Family composition", "Usage patterns" },
        DataSubjectCategories = new[] { "App users", "Family members" },
        Purpose = "Personalized meal planning and recipe suggestions",
        Recipients = new[] { "Google Gemini AI service (processor)", "Internal development team" },
        RetentionPeriod = "12 months for personalization, anonymized for model improvement",
        SecurityMeasures = "Data minimization, encryption, pseudonymization for AI processing",
        LastReviewed = DateTime.Parse("2024-01-15")
    };
}
```

---

*Last Updated: December 2024*  
*Data protection guide continuously updated to reflect evolving privacy regulations and best practices*
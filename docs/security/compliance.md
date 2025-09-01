# Compliance and Regulatory Requirements Guide

## Overview
Comprehensive guide for meeting regulatory compliance requirements in the MealPrep application, covering GDPR, CCPA, SOC 2, HIPAA considerations, and other relevant privacy and security standards.

## Compliance Framework

### Regulatory Standards Overview
The MealPrep application must comply with multiple regulatory frameworks due to its handling of personal data, health information, and operation across multiple jurisdictions.

### Primary Compliance Requirements
1. **GDPR (General Data Protection Regulation)** - EU privacy regulation
2. **CCPA (California Consumer Privacy Act)** - California privacy law
3. **SOC 2 Type II** - Security controls for service organizations
4. **COPPA (Children's Online Privacy Protection Act)** - If serving users under 13
5. **PCI DSS** - If processing payment information
6. **PIPEDA** - Canadian privacy legislation
7. **ISO 27001** - Information security management standard

---

## GDPR Compliance

### Legal Basis for Processing
```yaml
Legal Bases Under GDPR Article 6:
  Account Management:
    - Legal Basis: Contract (Article 6(1)(b))
    - Purpose: User account creation and management
    - Data: Email, name, password hash
    
  AI Meal Planning:
    - Legal Basis: Legitimate Interest (Article 6(1)(f))
    - Purpose: Personalized meal suggestions
    - Data: Dietary preferences, family member data
    - Balancing Test: User benefit vs. privacy impact
    
  Family Health Data:
    - Legal Basis: Consent (Article 6(1)(a))
    - Purpose: Health-conscious meal planning
    - Data: Allergies, health conditions, dietary restrictions
    - Special Category: Article 9(2)(a) - Explicit consent required
    
  Marketing Communications:
    - Legal Basis: Consent (Article 6(1)(a))
    - Purpose: Recipe recommendations, feature updates
    - Data: Email, usage patterns
```

### Data Subject Rights Implementation

#### Right of Access (Article 15)
```csharp
// Implementation of data subject access requests
public class DataSubjectAccessService
{
    private readonly IUserRepository _userRepository;
    private readonly IRecipeRepository _recipeRepository;
    private readonly IFamilyMemberRepository _familyRepository;
    private readonly IAuditLogRepository _auditRepository;

    public async Task<DataSubjectAccessResponse> GenerateDataExport(int userId)
    {
        // Compile all personal data for the user
        var userData = await _userRepository.GetUserDataAsync(userId);
        var recipes = await _recipeRepository.GetUserRecipesAsync(userId);
        var familyMembers = await _familyRepository.GetFamilyMembersAsync(userId);
        var auditLogs = await _auditRepository.GetUserActivityAsync(userId);
        
        return new DataSubjectAccessResponse
        {
            PersonalData = new
            {
                Account = userData,
                Recipes = recipes.Select(r => new
                {
                    r.Id,
                    r.Name,
                    r.Description,
                    r.CreatedDate,
                    r.IsPrivate
                }),
                FamilyMembers = familyMembers.Select(fm => new
                {
                    fm.Name,
                    fm.Age,
                    fm.DietaryRestrictions,
                    fm.Allergies,
                    fm.PreferredCuisines
                }),
                ActivityHistory = auditLogs.Select(log => new
                {
                    log.Action,
                    log.Timestamp,
                    log.IpAddress
                })
            },
            ProcessingPurposes = GetProcessingPurposes(),
            DataRetentionPeriods = GetRetentionPeriods(),
            ThirdPartySharing = GetThirdPartySharing(),
            GeneratedDate = DateTime.UtcNow
        };
    }

    private List<ProcessingPurpose> GetProcessingPurposes()
    {
        return new List<ProcessingPurpose>
        {
            new ProcessingPurpose
            {
                Purpose = "Account Management",
                LegalBasis = "Contract",
                DataCategories = new[] { "Contact information", "Account credentials" },
                RetentionPeriod = "Until account deletion + 30 days for legal compliance"
            },
            new ProcessingPurpose
            {
                Purpose = "AI Meal Planning",
                LegalBasis = "Legitimate Interest",
                DataCategories = new[] { "Dietary preferences", "Usage patterns" },
                RetentionPeriod = "3 years from last activity"
            },
            new ProcessingPurpose
            {
                Purpose = "Health-Conscious Recommendations",
                LegalBasis = "Explicit Consent",
                DataCategories = new[] { "Health conditions", "Allergies" },
                RetentionPeriod = "Until consent withdrawal"
            }
        };
    }
}
```

#### Right to Rectification (Article 16)
```csharp
public class DataRectificationService
{
    public async Task<bool> UpdatePersonalDataAsync(int userId, DataRectificationRequest request)
    {
        // Validate request
        if (!IsValidRectificationRequest(request))
            return false;

        // Log the rectification request
        await _auditService.LogDataRectificationAsync(userId, request);

        // Update the data
        switch (request.DataCategory)
        {
            case "Profile":
                await UpdateProfileDataAsync(userId, request.Updates);
                break;
            case "FamilyMember":
                await UpdateFamilyMemberDataAsync(userId, request.FamilyMemberId, request.Updates);
                break;
            case "HealthData":
                // Requires special handling for health data
                await UpdateHealthDataAsync(userId, request.Updates);
                break;
        }

        // Notify user of successful update
        await _notificationService.SendDataUpdateConfirmationAsync(userId);
        
        return true;
    }
}
```

#### Right to Erasure (Article 17)
```csharp
public class DataErasureService
{
    public async Task<ErasureResult> ProcessErasureRequestAsync(int userId, ErasureRequest request)
    {
        var result = new ErasureResult { UserId = userId, RequestDate = DateTime.UtcNow };

        try
        {
            // 1. Validate legal grounds for erasure
            var legalGrounds = await ValidateErasureGroundsAsync(userId, request);
            if (!legalGrounds.IsValid)
            {
                result.Status = ErasureStatus.Denied;
                result.Reason = legalGrounds.Reason;
                return result;
            }

            // 2. Check for legal obligations to retain data
            var retentionRequirements = await CheckRetentionRequirementsAsync(userId);
            if (retentionRequirements.MustRetain)
            {
                result.Status = ErasureStatus.PartiallyCompleted;
                result.RetainedData = retentionRequirements.Categories;
            }

            // 3. Perform data deletion
            await DeleteUserDataAsync(userId, request.Categories);
            
            // 4. Notify third parties if data was shared
            await NotifyThirdPartiesOfErasureAsync(userId, request.Categories);

            result.Status = result.Status == ErasureStatus.PartiallyCompleted 
                ? ErasureStatus.PartiallyCompleted 
                : ErasureStatus.Completed;
                
        }
        catch (Exception ex)
        {
            result.Status = ErasureStatus.Failed;
            result.Error = ex.Message;
            _logger.LogError(ex, "Failed to process erasure request for user {UserId}", userId);
        }

        return result;
    }

    private async Task DeleteUserDataAsync(int userId, List<string> categories)
    {
        // Delete in order to maintain referential integrity
        if (categories.Contains("ActivityLogs"))
            await _auditRepository.DeleteUserActivityAsync(userId);
            
        if (categories.Contains("Recipes"))
            await _recipeRepository.DeleteUserRecipesAsync(userId);
            
        if (categories.Contains("FamilyMembers"))
            await _familyRepository.DeleteFamilyMembersAsync(userId);
            
        if (categories.Contains("Profile"))
        {
            // Mark account as deleted rather than hard delete for audit purposes
            await _userRepository.MarkAccountDeletedAsync(userId);
        }
    }
}
```

### Data Protection Impact Assessment (DPIA)

#### DPIA for AI Meal Planning Feature
```markdown
# Data Protection Impact Assessment - AI Meal Planning

## Processing Overview
- **Purpose**: Generate personalized meal suggestions using AI
- **Data Types**: Dietary preferences, family member data, health conditions
- **Technology**: Google Gemini AI API
- **Data Volume**: ~10,000 users, 50,000 family member profiles

## Risk Assessment

### High Risks Identified
1. **Sensitive Health Data Processing**
   - Risk: Unauthorized disclosure of health conditions
   - Likelihood: Medium
   - Impact: High
   - Mitigation: Encryption, access controls, data minimization

2. **AI Bias and Discrimination**
   - Risk: Biased meal suggestions based on protected characteristics
   - Likelihood: Medium
   - Impact: Medium
   - Mitigation: Bias testing, diverse training data, human oversight

3. **Third-Party Data Sharing**
   - Risk: Google AI service access to personal data
   - Likelihood: High (by design)
   - Impact: Medium
   - Mitigation: Data Processing Agreement, encryption in transit

### Mitigation Measures
1. **Technical Safeguards**
   - End-to-end encryption for health data
   - Pseudonymization of identifiers sent to AI service
   - Regular security assessments

2. **Organizational Measures**
   - Staff training on health data handling
   - Incident response procedures
   - Regular compliance audits

3. **Legal Safeguards**
   - Clear consent mechanisms
   - Privacy notices in plain language
   - Data Processing Agreements with vendors

## Compliance Recommendation
Proceed with implementation subject to:
- Implementation of all identified mitigation measures
- Additional consent flow for health data processing
- Quarterly review of AI bias metrics
```

### Cookie Compliance and Consent Management

#### Cookie Classification and Management
```typescript
// Cookie consent management implementation
interface CookieCategory {
  id: string;
  name: string;
  description: string;
  required: boolean;
  cookies: Cookie[];
}

interface Cookie {
  name: string;
  purpose: string;
  duration: string;
  provider: string;
  type: 'functional' | 'analytics' | 'marketing' | 'personalization';
}

export class CookieConsentManager {
  private consentData: Map<string, boolean> = new Map();
  
  getCookieCategories(): CookieCategory[] {
    return [
      {
        id: 'necessary',
        name: 'Strictly Necessary',
        description: 'Essential for basic website functionality and security',
        required: true,
        cookies: [
          {
            name: 'mealprep_session',
            purpose: 'Maintains user session and authentication state',
            duration: 'Session',
            provider: 'MealPrep',
            type: 'functional'
          },
          {
            name: 'csrf_token',
            purpose: 'Prevents cross-site request forgery attacks',
            duration: 'Session',
            provider: 'MealPrep',
            type: 'functional'
          }
        ]
      },
      {
        id: 'functional',
        name: 'Functional',
        description: 'Enhance website functionality and personalization',
        required: false,
        cookies: [
          {
            name: 'user_preferences',
            purpose: 'Stores user interface preferences and settings',
            duration: '1 year',
            provider: 'MealPrep',
            type: 'personalization'
          },
          {
            name: 'family_context',
            purpose: 'Remembers selected family member for meal planning',
            duration: '30 days',
            provider: 'MealPrep',
            type: 'personalization'
          }
        ]
      },
      {
        id: 'analytics',
        name: 'Analytics',
        description: 'Help us understand how users interact with our website',
        required: false,
        cookies: [
          {
            name: '_ga',
            purpose: 'Google Analytics - tracks user sessions and behavior',
            duration: '2 years',
            provider: 'Google',
            type: 'analytics'
          },
          {
            name: 'mealprep_analytics',
            purpose: 'Internal analytics for feature usage tracking',
            duration: '1 year',
            provider: 'MealPrep',
            type: 'analytics'
          }
        ]
      },
      {
        id: 'marketing',
        name: 'Marketing',
        description: 'Used for targeted advertising and marketing campaigns',
        required: false,
        cookies: [
          {
            name: '_fbp',
            purpose: 'Facebook Pixel for conversion tracking',
            duration: '90 days',
            provider: 'Facebook',
            type: 'marketing'
          }
        ]
      }
    ];
  }

  updateConsent(categoryId: string, granted: boolean): void {
    this.consentData.set(categoryId, granted);
    this.applyConsentSettings();
    this.saveConsentRecord(categoryId, granted);
  }

  private applyConsentSettings(): void {
    // Enable/disable cookies based on consent
    if (!this.consentData.get('analytics')) {
      this.deleteCookiesByCategory('analytics');
      this.disableGoogleAnalytics();
    }

    if (!this.consentData.get('marketing')) {
      this.deleteCookiesByCategory('marketing');
      this.disableFacebookPixel();
    }
  }

  private saveConsentRecord(categoryId: string, granted: boolean): void {
    const consentRecord = {
      userId: this.getCurrentUserId(),
      categoryId,
      granted,
      timestamp: new Date().toISOString(),
      ipAddress: this.getUserIpAddress(),
      userAgent: navigator.userAgent
    };

    // Save to backend for compliance auditing
    fetch('/api/privacy/consent', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(consentRecord)
    });
  }
}
```

---

## CCPA Compliance

### Consumer Rights Implementation

#### Right to Know and Delete
```csharp
public class CCPAComplianceService
{
    public async Task<CCPADataResponse> ProcessConsumerDataRequestAsync(CCPARequest request)
    {
        // Verify consumer identity
        var identity = await VerifyConsumerIdentityAsync(request);
        if (!identity.IsVerified)
        {
            return new CCPADataResponse 
            { 
                Status = "Denied", 
                Reason = "Unable to verify consumer identity" 
            };
        }

        switch (request.RequestType)
        {
            case CCPARequestType.RightToKnow:
                return await ProcessRightToKnowAsync(identity.UserId);
            
            case CCPARequestType.RightToDelete:
                return await ProcessRightToDeleteAsync(identity.UserId, request.Categories);
            
            case CCPARequestType.RightToOptOut:
                return await ProcessOptOutRequestAsync(identity.UserId);
                
            default:
                throw new ArgumentException("Invalid CCPA request type");
        }
    }

    private async Task<CCPADataResponse> ProcessRightToKnowAsync(int userId)
    {
        var personalInfo = await GetPersonalInformationAsync(userId);
        
        return new CCPADataResponse
        {
            Status = "Completed",
            Categories = new CCPADataCategories
            {
                Identifiers = new[]
                {
                    "Email address",
                    "Name",
                    "Account ID"
                },
                CommercialInformation = new[]
                {
                    "Recipe preferences",
                    "Subscription status",
                    "Purchase history"
                },
                BiometricInformation = new string[0], // Not collected
                InternetActivity = new[]
                {
                    "Website usage patterns",
                    "Recipe search history",
                    "Feature usage analytics"
                },
                GeolocationData = new string[0], // Not collected
                SensoryData = new string[0], // Not collected
                ProfessionalInformation = new string[0], // Not collected
                EducationInformation = new string[0], // Not collected
                InferencesDrawn = new[]
                {
                    "Dietary preferences",
                    "Cooking skill level",
                    "Family meal patterns"
                }
            },
            Sources = new[]
            {
                "Directly from consumer",
                "Consumer device interactions",
                "Third-party analytics services"
            },
            BusinessPurposes = new[]
            {
                "Provide meal planning services",
                "Improve AI recommendations",
                "Customer support",
                "Security and fraud prevention"
            },
            ThirdParties = new[]
            {
                "Google (AI processing)",
                "Analytics providers",
                "Cloud infrastructure providers"
            }
        };
    }
}
```

#### Do Not Sell My Personal Information
```csharp
public class DoNotSellService
{
    public async Task ProcessOptOutRequestAsync(int userId)
    {
        // Record opt-out preference
        await _privacyRepository.SetDoNotSellFlagAsync(userId, true);
        
        // Stop all data sharing for advertising purposes
        await _marketingService.RemoveFromAudiencesAsync(userId);
        
        // Notify third parties of opt-out
        await NotifyPartnersOfOptOutAsync(userId);
        
        // Update consent management
        await _consentService.UpdateMarketingConsentAsync(userId, false);
        
        _logger.LogInformation("Processed CCPA opt-out request for user {UserId}", userId);
    }

    public async Task<bool> IsOptedOutAsync(int userId)
    {
        return await _privacyRepository.GetDoNotSellFlagAsync(userId);
    }
}
```

---

## SOC 2 Type II Compliance

### Security Controls Implementation

#### Access Controls (CC6.1)
```csharp
public class AccessControlAudit
{
    public async Task<AccessControlReport> GenerateAccessControlReportAsync()
    {
        return new AccessControlReport
        {
            UserAccessReview = await ReviewUserAccessAsync(),
            PrivilegedAccessReview = await ReviewPrivilegedAccessAsync(),
            ServiceAccountReview = await ReviewServiceAccountsAsync(),
            AccessProvisioningProcess = GetAccessProvisioningControls(),
            AccessTerminationProcess = GetAccessTerminationControls(),
            RegularAccessReviews = await GetAccessReviewHistoryAsync()
        };
    }

    private async Task<UserAccessReview> ReviewUserAccessAsync()
    {
        var users = await _userRepository.GetAllUsersAsync();
        var reviews = new List<UserAccessItem>();

        foreach (var user in users)
        {
            var lastLogin = await _auditRepository.GetLastLoginAsync(user.Id);
            var permissions = await _authService.GetUserPermissionsAsync(user.Id);
            
            reviews.Add(new UserAccessItem
            {
                UserId = user.Id,
                Email = user.Email,
                LastLogin = lastLogin,
                DaysSinceLastLogin = (DateTime.UtcNow - lastLogin).Days,
                Permissions = permissions,
                RequiresReview = (DateTime.UtcNow - lastLogin).Days > 90
            });
        }

        return new UserAccessReview
        {
            TotalUsers = reviews.Count,
            ActiveUsers = reviews.Count(r => r.DaysSinceLastLogin <= 30),
            InactiveUsers = reviews.Count(r => r.DaysSinceLastLogin > 90),
            RequireAccessReview = reviews.Where(r => r.RequiresReview).ToList()
        };
    }
}
```

#### System Monitoring (CC7.1)
```csharp
public class SystemMonitoringControls
{
    public async Task<MonitoringControlsReport> GenerateMonitoringReportAsync()
    {
        return new MonitoringControlsReport
        {
            SecurityEventMonitoring = await GetSecurityEventMonitoringAsync(),
            PerformanceMonitoring = await GetPerformanceMonitoringAsync(),
            AvailabilityMonitoring = await GetAvailabilityMonitoringAsync(),
            LoggingConfiguration = GetLoggingConfigurationAsync(),
            AlertingConfiguration = GetAlertingConfigurationAsync(),
            IncidentResponseMetrics = await GetIncidentResponseMetricsAsync()
        };
    }

    private async Task<SecurityEventMonitoring> GetSecurityEventMonitoringAsync()
    {
        return new SecurityEventMonitoring
        {
            FailedLoginAttempts = await _securityRepository.GetFailedLoginCountAsync(TimeSpan.FromDays(30)),
            UnauthorizedAccessAttempts = await _securityRepository.GetUnauthorizedAccessCountAsync(TimeSpan.FromDays(30)),
            SuspiciousActivities = await _securityRepository.GetSuspiciousActivitiesAsync(TimeSpan.FromDays(30)),
            SecurityAlertsGenerated = await _alertRepository.GetSecurityAlertCountAsync(TimeSpan.FromDays(30)),
            SecurityAlertsResolved = await _alertRepository.GetResolvedSecurityAlertCountAsync(TimeSpan.FromDays(30))
        };
    }
}
```

### Data Protection Controls

#### Encryption at Rest and in Transit
```yaml
# SOC 2 Encryption Controls Documentation
Encryption Standards:
  Data at Rest:
    Database: AES-256 encryption using Transparent Data Encryption (TDE)
    File Storage: AES-256 encryption for all stored files
    Backups: AES-256 encryption for all backup files
    Key Management: Azure Key Vault with HSM protection
    
  Data in Transit:
    API Communications: TLS 1.3 minimum
    Database Connections: TLS 1.2 minimum with certificate validation
    Internal Services: mTLS for service-to-service communication
    Client Connections: HTTPS only with HSTS headers
    
  Key Management:
    Rotation Policy: Keys rotated every 90 days
    Access Control: Role-based access to encryption keys
    Audit Logging: All key access logged and monitored
    Backup: Keys backed up to secure offline storage
```

---

## HIPAA Considerations

### Business Associate Agreement Template
```markdown
# Business Associate Agreement - Health Data Processing

## Scope
While MealPrep is not a covered entity under HIPAA, when processing health information 
for healthcare providers or patients, we implement HIPAA-aligned safeguards.

### Permitted Uses and Disclosures
- Use PHI solely for meal planning and nutritional guidance services
- Disclose PHI only as required for service provision
- No marketing use of PHI without specific authorization

### Safeguards Required
1. **Administrative Safeguards**
   - Designated security officer
   - Workforce training on PHI handling
   - Access management procedures
   - Incident response procedures

2. **Physical Safeguards**
   - Secure data centers with access controls
   - Workstation security measures
   - Device and media controls

3. **Technical Safeguards**
   - Access controls and user authentication
   - Audit logs and monitoring
   - Data integrity controls
   - Transmission security (encryption)

### Breach Notification
- Report any suspected breach within 24 hours
- Provide detailed incident information
- Assist with breach assessment and notification
```

---

## Audit and Reporting

### Compliance Monitoring Dashboard
```typescript
interface ComplianceMetrics {
  gdprCompliance: {
    dataSubjectRequestsReceived: number;
    dataSubjectRequestsCompleted: number;
    averageResponseTime: number;
    consentWithdrawals: number;
    dataBreaches: number;
  };
  
  ccpaCompliance: {
    consumerRequestsReceived: number;
    optOutRequests: number;
    dataSellingOptOuts: number;
    requestProcessingTime: number;
  };
  
  soc2Compliance: {
    securityIncidents: number;
    vulnerabilitiesRemediated: number;
    accessReviewsCompleted: number;
    controlTestingResults: number;
  };
  
  generalCompliance: {
    privacyPolicyUpdates: number;
    staffTrainingCompletion: number;
    vendorAssessments: number;
    auditFindings: number;
  };
}

export class ComplianceMonitor {
  async generateComplianceReport(): Promise<ComplianceReport> {
    const metrics = await this.collectComplianceMetrics();
    const riskAssessment = await this.performRiskAssessment();
    const recommendations = this.generateRecommendations(metrics, riskAssessment);

    return {
      reportPeriod: this.getCurrentReportPeriod(),
      overallComplianceScore: this.calculateComplianceScore(metrics),
      metrics,
      riskAssessment,
      recommendations,
      nextAuditDate: this.getNextAuditDate(),
      generatedAt: new Date().toISOString()
    };
  }

  private calculateComplianceScore(metrics: ComplianceMetrics): number {
    // Weighted scoring based on regulatory requirements
    const scores = {
      gdpr: this.scoreGDPRCompliance(metrics.gdprCompliance),
      ccpa: this.scoreCCPACompliance(metrics.ccpaCompliance),
      soc2: this.scoreSOC2Compliance(metrics.soc2Compliance),
      general: this.scoreGeneralCompliance(metrics.generalCompliance)
    };

    // Weighted average (GDPR and SOC2 have higher weights)
    return (
      scores.gdpr * 0.3 +
      scores.ccpa * 0.2 +
      scores.soc2 * 0.3 +
      scores.general * 0.2
    );
  }
}
```

### Regular Compliance Auditing
```bash
#!/bin/bash
# scripts/compliance/monthly-compliance-audit.sh

echo "Starting monthly compliance audit..."

# Check data retention compliance
echo "Checking data retention policies..."
node scripts/compliance/check-data-retention.js

# Verify encryption standards
echo "Verifying encryption compliance..."
node scripts/compliance/check-encryption.js

# Review access controls
echo "Reviewing access controls..."
node scripts/compliance/audit-access-controls.js

# Check privacy policy compliance
echo "Checking privacy policy compliance..."
node scripts/compliance/verify-privacy-notices.js

# Generate compliance report
echo "Generating compliance report..."
node scripts/compliance/generate-report.js

echo "Monthly compliance audit completed"
```

---

## Staff Training and Awareness

### Privacy Training Program
```markdown
# Privacy and Compliance Training Requirements

## All Staff Training (Annual)
- Data protection principles and regulations
- Company privacy policies and procedures
- Incident reporting requirements
- Basic security awareness

## Developer Training (Bi-annual)
- Privacy by design principles
- Secure coding practices for privacy
- Data minimization techniques
- Consent management implementation

## Support Staff Training (Quarterly)
- Handling data subject requests
- Customer privacy inquiries
- Escalation procedures
- Documentation requirements

## Management Training (Annual)
- Compliance oversight responsibilities
- Risk assessment and management
- Vendor management and due diligence
- Breach response coordination

## Training Tracking
- Completion certificates maintained
- Regular assessment and testing
- Continuous education on regulatory updates
- Performance metrics and improvement plans
```

---

*Last Updated: December 2024*  
*Compliance guide continuously updated with regulatory changes and requirements*
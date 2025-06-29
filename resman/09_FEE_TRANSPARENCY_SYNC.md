# Fee Transparency Sync: ResMan → DomIQ

## Feature Overview
Centralized fee management system that automatically pulls application fees, admin fees, pet fees, and monthly add-on fees from ResMan to ensure every quote, chat response, and application shows exact, up-to-date costs with complete transparency.

## What It Delivers for On-site Teams
- **Complete Fee Transparency**: All costs clearly displayed to prospects
- **Real-time Accuracy**: Fees always match current ResMan rates
- **Consistent Pricing**: Same fees shown across all channels
- **Automated Updates**: No manual fee management required
- **Compliance Ready**: Full disclosure of all costs
- **Trust Building**: No hidden fees or surprises

## Data Flow Architecture

### 1. Fee Sync Flow
```
ResMan Fee Tables → Fee Extraction → Validation → DomIQ Cache → AI Knowledge → Quote Generation
```

### 2. Real-time Fee Display Flow
```
Prospect Query → Fee Lookup → Current Rates → Transparent Display → Trust Building
```

## Implementation Requirements

### No Database Required
This system operates with in-memory caching and direct ResMan API integration. No persistent storage needed.

### API Endpoints to Create

#### 1. Fee Sync Trigger API
```typescript
// POST /api/resman/fees/sync
interface FeeSyncRequest {
  force_refresh?: boolean; // Force immediate sync regardless of schedule
  fee_types?: string[]; // Specific fee types to sync (optional)
  properties?: string[]; // Specific properties to sync (optional)
}

interface FeeSyncResponse {
  success: boolean;
  sync_id: string;
  properties_processed: number;
  fee_types_processed: number;
  processing_time_ms: number;
  sync_summary: {
    application_fees: number;
    admin_fees: number;
    pet_fees: number;
    addon_fees: number;
    fee_updates: number;
  };
  errors?: string[];
}
```

#### 2. Fee Status API
```typescript
// GET /api/resman/fees/status
interface FeeStatusResponse {
  last_sync: {
    timestamp: string;
    success: boolean;
    properties_processed: number;
    fee_types_processed: number;
    processing_time_ms: number;
  };
  next_scheduled_sync: string;
  sync_schedule: {
    frequency: string; // "hourly", "daily", "on_change"
    last_check: string;
  };
  current_cache_status: {
    total_fees: number;
    properties_with_fees: number;
    cache_age_minutes: number;
    is_stale: boolean; // True if cache is older than 1 hour
  };
}
```

#### 3. Fee Query API
```typescript
// GET /api/resman/fees/query
interface FeeQueryRequest {
  property_id?: string;
  fee_types?: string[]; // 'application', 'admin', 'pet', 'addon'
  unit_type?: string;
  pet_type?: string; // 'dog', 'cat', 'other'
  addon_types?: string[]; // 'parking', 'storage', 'utilities', etc.
  include_breakdown?: boolean;
  include_conditions?: boolean;
}

interface FeeQueryResponse {
  fees: Array<{
    fee_id: string;
    property_id: string;
    property_name: string;
    fee_type: 'application' | 'admin' | 'pet' | 'addon';
    fee_name: string;
    amount: number;
    currency: string;
    fee_category: string;
    description: string;
    conditions: Array<{
      condition_type: string;
      condition_value: string;
      condition_description: string;
    }>;
    is_required: boolean;
    is_refundable: boolean;
    payment_timing: 'upfront' | 'monthly' | 'upon_move_in' | 'upon_approval';
    last_updated: string;
  }>;
  summary: {
    total_application_fees: number;
    total_admin_fees: number;
    total_pet_fees: number;
    total_addon_fees: number;
    estimated_total: number;
    breakdown_by_type: Record<string, number>;
  };
}
```

#### 4. Fee Calculator API
```typescript
// POST /api/resman/fees/calculate
interface FeeCalculationRequest {
  property_id: string;
  unit_type?: string;
  pets?: Array<{
    type: string; // 'dog', 'cat', 'other'
    count: number;
    weight?: number;
    breed?: string;
  }>;
  addons?: Array<{
    type: string;
    quantity?: number;
    duration_months?: number;
  }>;
  lease_term_months?: number;
  include_breakdown?: boolean;
}

interface FeeCalculationResponse {
  calculation_id: string;
  property_id: string;
  total_fees: number;
  breakdown: {
    application_fees: Array<{
      fee_name: string;
      amount: number;
      description: string;
      is_required: boolean;
    }>;
    admin_fees: Array<{
      fee_name: string;
      amount: number;
      description: string;
      payment_timing: string;
    }>;
    pet_fees: Array<{
      fee_name: string;
      amount: number;
      pet_type: string;
      description: string;
    }>;
    addon_fees: Array<{
      fee_name: string;
      amount: number;
      addon_type: string;
      monthly_cost: number;
      total_cost: number;
    }>;
  };
  summary: {
    upfront_costs: number;
    monthly_addons: number;
    total_first_month: number;
    refundable_amount: number;
    non_refundable_amount: number;
  };
  disclaimers: string[];
  calculation_timestamp: string;
}
```

## ResMan Fee Extraction

### 1. Fee Data Extraction
```typescript
class ResManFeeExtractor {
  async extractFeeData(propertyIds?: string[]): Promise<FeeData> {
    try {
      const properties = propertyIds || await this.getAllPropertyIds();
      
      const feeData: FeeData = {
        properties: {},
        extraction_timestamp: new Date().toISOString()
      };
      
      for (const propertyId of properties) {
        const propertyFees = await this.extractPropertyFees(propertyId);
        feeData.properties[propertyId] = propertyFees;
      }
      
      return feeData;
      
    } catch (error) {
      throw new Error(`Failed to extract fee data: ${error.message}`);
    }
  }

  private async extractPropertyFees(propertyId: string): Promise<PropertyFees> {
    const [applicationFees, adminFees, petFees, addonFees] = await Promise.all([
      this.resManAPI.getApplicationFees({ property_id: propertyId }),
      this.resManAPI.getAdminFees({ property_id: propertyId }),
      this.resManAPI.getPetFees({ property_id: propertyId }),
      this.resManAPI.getAddonFees({ property_id: propertyId })
    ]);

    return {
      application_fees: this.mapApplicationFees(applicationFees),
      admin_fees: this.mapAdminFees(adminFees),
      pet_fees: this.mapPetFees(petFees),
      addon_fees: this.mapAddonFees(addonFees),
      property_info: await this.getPropertyInfo(propertyId)
    };
  }

  private mapApplicationFees(resmanFees: any[]): ApplicationFee[] {
    return resmanFees.map(fee => ({
      fee_id: fee.id,
      fee_name: fee.name,
      amount: fee.amount,
      currency: fee.currency || 'USD',
      fee_category: 'application',
      description: fee.description,
      conditions: this.mapConditions(fee.conditions),
      is_required: fee.is_required,
      is_refundable: fee.is_refundable,
      payment_timing: 'upfront',
      last_updated: fee.last_updated
    }));
  }

  private mapAdminFees(resmanFees: any[]): AdminFee[] {
    return resmanFees.map(fee => ({
      fee_id: fee.id,
      fee_name: fee.name,
      amount: fee.amount,
      currency: fee.currency || 'USD',
      fee_category: 'admin',
      description: fee.description,
      conditions: this.mapConditions(fee.conditions),
      is_required: fee.is_required,
      is_refundable: fee.is_refundable,
      payment_timing: fee.payment_timing || 'upon_approval',
      last_updated: fee.last_updated
    }));
  }

  private mapPetFees(resmanFees: any[]): PetFee[] {
    return resmanFees.map(fee => ({
      fee_id: fee.id,
      fee_name: fee.name,
      amount: fee.amount,
      currency: fee.currency || 'USD',
      fee_category: 'pet',
      description: fee.description,
      pet_type: fee.pet_type,
      pet_conditions: this.mapPetConditions(fee.pet_conditions),
      is_required: fee.is_required,
      is_refundable: fee.is_refundable,
      payment_timing: fee.payment_timing || 'upfront',
      last_updated: fee.last_updated
    }));
  }

  private mapAddonFees(resmanFees: any[]): AddonFee[] {
    return resmanFees.map(fee => ({
      fee_id: fee.id,
      fee_name: fee.name,
      amount: fee.amount,
      currency: fee.currency || 'USD',
      fee_category: 'addon',
      description: fee.description,
      addon_type: fee.addon_type,
      monthly_cost: fee.monthly_cost,
      setup_cost: fee.setup_cost,
      conditions: this.mapConditions(fee.conditions),
      is_required: fee.is_required,
      payment_timing: fee.payment_timing || 'monthly',
      last_updated: fee.last_updated
    }));
  }

  private mapConditions(resmanConditions: any[]): FeeCondition[] {
    return resmanConditions.map(condition => ({
      condition_type: condition.type,
      condition_value: condition.value,
      condition_description: condition.description
    }));
  }

  private mapPetConditions(resmanConditions: any[]): PetCondition[] {
    return resmanConditions.map(condition => ({
      pet_type: condition.pet_type,
      weight_limit: condition.weight_limit,
      breed_restrictions: condition.breed_restrictions,
      additional_fees: condition.additional_fees
    }));
  }
}
```

### 2. Fee Validation Engine
```typescript
class FeeDataValidator {
  validateFeeData(feeData: FeeData): ValidationResult {
    const errors: string[] = [];
    const warnings: string[] = [];
    const validFees: ValidatedFees = {
      properties: {}
    };

    for (const [propertyId, propertyFees] of Object.entries(feeData.properties)) {
      const propertyValidation = this.validatePropertyFees(propertyId, propertyFees);
      
      if (propertyValidation.isValid) {
        validFees.properties[propertyId] = propertyValidation.fees;
      } else {
        errors.push(`Property ${propertyId}: ${propertyValidation.errors.join(', ')}`);
      }
      
      if (propertyValidation.warnings.length > 0) {
        warnings.push(`Property ${propertyId}: ${propertyValidation.warnings.join(', ')}`);
      }
    }

    return {
      valid_fees: validFees,
      total_properties: Object.keys(feeData.properties).length,
      valid_properties: Object.keys(validFees.properties).length,
      error_count: errors.length,
      warning_count: warnings.length,
      errors,
      warnings
    };
  }

  private validatePropertyFees(propertyId: string, propertyFees: PropertyFees): PropertyFeeValidation {
    const errors: string[] = [];
    const warnings: string[] = [];
    const validFees: PropertyFees = {
      application_fees: [],
      admin_fees: [],
      pet_fees: [],
      addon_fees: [],
      property_info: propertyFees.property_info
    };

    // Validate application fees
    propertyFees.application_fees.forEach(fee => {
      const feeValidation = this.validateFee(fee, 'application');
      if (feeValidation.isValid) {
        validFees.application_fees.push(fee);
      } else {
        errors.push(`Application fee ${fee.fee_name}: ${feeValidation.errors.join(', ')}`);
      }
    });

    // Validate admin fees
    propertyFees.admin_fees.forEach(fee => {
      const feeValidation = this.validateFee(fee, 'admin');
      if (feeValidation.isValid) {
        validFees.admin_fees.push(fee);
      } else {
        errors.push(`Admin fee ${fee.fee_name}: ${feeValidation.errors.join(', ')}`);
      }
    });

    // Validate pet fees
    propertyFees.pet_fees.forEach(fee => {
      const feeValidation = this.validatePetFee(fee);
      if (feeValidation.isValid) {
        validFees.pet_fees.push(fee);
      } else {
        errors.push(`Pet fee ${fee.fee_name}: ${feeValidation.errors.join(', ')}`);
      }
    });

    // Validate addon fees
    propertyFees.addon_fees.forEach(fee => {
      const feeValidation = this.validateAddonFee(fee);
      if (feeValidation.isValid) {
        validFees.addon_fees.push(fee);
      } else {
        errors.push(`Addon fee ${fee.fee_name}: ${feeValidation.errors.join(', ')}`);
      }
    });

    // Business logic validation
    if (validFees.application_fees.length === 0) {
      warnings.push('No application fees found');
    }

    if (validFees.admin_fees.length === 0) {
      warnings.push('No admin fees found');
    }

    return {
      isValid: errors.length === 0,
      fees: validFees,
      errors,
      warnings
    };
  }

  private validateFee(fee: any, feeType: string): FeeValidation {
    const errors: string[] = [];
    const warnings: string[] = [];

    // Required field validation
    if (!fee.fee_id) errors.push('Missing fee ID');
    if (!fee.fee_name) errors.push('Missing fee name');
    if (fee.amount < 0) errors.push('Invalid fee amount');
    if (!fee.currency) errors.push('Missing currency');

    // Business logic validation
    if (fee.amount > 10000) {
      warnings.push('Fee amount seems unusually high');
    }

    if (fee.is_refundable && feeType === 'application') {
      warnings.push('Application fees are typically non-refundable');
    }

    return {
      isValid: errors.length === 0,
      errors,
      warnings
    };
  }
}
```

## Fee Processing and Caching

### 1. Fee Data Processor
```typescript
class FeeDataProcessor {
  private cache = new Map<string, any>();
  private lastUpdate = null;

  async processAndCacheFeeData(feeData: ValidatedFees): Promise<void> {
    try {
      // Process fees into optimized format for AI queries
      const processedData = this.processFees(feeData);
      
      // Update cache
      this.cache.set('fees', processedData);
      this.cache.set('fee_calculator', this.createFeeCalculator(processedData));
      this.cache.set('fee_search', this.createFeeSearchIndex(processedData));
      this.cache.set('fee_summaries', this.createFeeSummaries(processedData));
      
      this.lastUpdate = new Date();
      
      console.log(`Processed and cached fees for ${Object.keys(feeData.properties).length} properties`);
      
    } catch (error) {
      throw new Error(`Failed to process fee data: ${error.message}`);
    }
  }

  private processFees(feeData: ValidatedFees): ProcessedFeeData {
    return {
      by_property: feeData.properties,
      by_type: this.groupByFeeType(feeData),
      by_category: this.groupByCategory(feeData),
      calculator: this.createFeeCalculator(feeData),
      search_index: this.createFeeSearchIndex(feeData)
    };
  }

  private createFeeCalculator(feeData: ValidatedFees): FeeCalculator {
    return {
      calculateTotalFees: (request: FeeCalculationRequest) => {
        return this.calculateFees(request, feeData);
      },
      getFeeBreakdown: (propertyId: string, feeTypes: string[]) => {
        return this.getFeeBreakdown(propertyId, feeTypes, feeData);
      },
      estimateMonthlyCosts: (propertyId: string, addons: string[]) => {
        return this.estimateMonthlyCosts(propertyId, addons, feeData);
      }
    };
  }

  private calculateFees(request: FeeCalculationRequest, feeData: ValidatedFees): FeeCalculation {
    const propertyFees = feeData.properties[request.property_id];
    if (!propertyFees) {
      throw new Error(`No fees found for property ${request.property_id}`);
    }

    const calculation: FeeCalculation = {
      calculation_id: this.generateCalculationId(),
      property_id: request.property_id,
      total_fees: 0,
      breakdown: {
        application_fees: [],
        admin_fees: [],
        pet_fees: [],
        addon_fees: []
      },
      summary: {
        upfront_costs: 0,
        monthly_addons: 0,
        total_first_month: 0,
        refundable_amount: 0,
        non_refundable_amount: 0
      },
      disclaimers: [],
      calculation_timestamp: new Date().toISOString()
    };

    // Calculate application fees
    propertyFees.application_fees.forEach(fee => {
      if (this.feeApplies(fee, request)) {
        calculation.breakdown.application_fees.push({
          fee_name: fee.fee_name,
          amount: fee.amount,
          description: fee.description,
          is_required: fee.is_required
        });
        calculation.total_fees += fee.amount;
        calculation.summary.upfront_costs += fee.amount;
        if (fee.is_refundable) {
          calculation.summary.refundable_amount += fee.amount;
        } else {
          calculation.summary.non_refundable_amount += fee.amount;
        }
      }
    });

    // Calculate admin fees
    propertyFees.admin_fees.forEach(fee => {
      if (this.feeApplies(fee, request)) {
        calculation.breakdown.admin_fees.push({
          fee_name: fee.fee_name,
          amount: fee.amount,
          description: fee.description,
          payment_timing: fee.payment_timing
        });
        calculation.total_fees += fee.amount;
        calculation.summary.upfront_costs += fee.amount;
      }
    });

    // Calculate pet fees
    if (request.pets) {
      request.pets.forEach(pet => {
        const petFees = propertyFees.pet_fees.filter(fee => 
          fee.pet_type === pet.type && this.petFeeApplies(fee, pet)
        );
        
        petFees.forEach(fee => {
          calculation.breakdown.pet_fees.push({
            fee_name: fee.fee_name,
            amount: fee.amount * pet.count,
            pet_type: pet.type,
            description: fee.description
          });
          calculation.total_fees += fee.amount * pet.count;
          calculation.summary.upfront_costs += fee.amount * pet.count;
        });
      });
    }

    // Calculate addon fees
    if (request.addons) {
      request.addons.forEach(addon => {
        const addonFees = propertyFees.addon_fees.filter(fee => 
          fee.addon_type === addon.type
        );
        
        addonFees.forEach(fee => {
          const quantity = addon.quantity || 1;
          const duration = addon.duration_months || request.lease_term_months || 12;
          const totalCost = fee.setup_cost + (fee.monthly_cost * duration);
          
          calculation.breakdown.addon_fees.push({
            fee_name: fee.fee_name,
            amount: fee.setup_cost,
            addon_type: addon.type,
            monthly_cost: fee.monthly_cost,
            total_cost: totalCost
          });
          
          calculation.total_fees += fee.setup_cost;
          calculation.summary.upfront_costs += fee.setup_cost;
          calculation.summary.monthly_addons += fee.monthly_cost * quantity;
        });
      });
    }

    calculation.summary.total_first_month = calculation.summary.upfront_costs + calculation.summary.monthly_addons;

    return calculation;
  }

  getCachedData(): CachedFeeData {
    return {
      fees: this.cache.get('fees'),
      calculator: this.cache.get('fee_calculator'),
      search_index: this.cache.get('fee_search'),
      summaries: this.cache.get('fee_summaries'),
      last_update: this.lastUpdate,
      cache_age_minutes: this.lastUpdate ? 
        Math.floor((Date.now() - this.lastUpdate.getTime()) / 60000) : null
    };
  }
}
```

## AI Integration

### 1. AI Fee Knowledge Update
```typescript
class AIFeeKnowledgeUpdater {
  async updateFeeKnowledge(feeData: ValidatedFees): Promise<void> {
    try {
      // Update AI's understanding of current fees
      await this.aiService.updateContext({
        type: 'fee_transparency',
        data: {
          properties: this.createPropertyFeeSummary(feeData),
          fee_types: this.createFeeTypeSummary(feeData),
          common_fees: this.identifyCommonFees(feeData),
          fee_ranges: this.calculateFeeRanges(feeData)
        },
        timestamp: new Date().toISOString()
      });

      // Update AI constraints for fee transparency
      await this.aiService.updateConstraints({
        type: 'fee_disclosure',
        constraints: {
          always_show_fees: true,
          include_breakdown: true,
          show_refundable_status: true,
          disclose_all_costs: true,
          no_hidden_fees: true
        }
      });

      console.log('AI fee knowledge updated with current fee data');
      
    } catch (error) {
      throw new Error(`Failed to update AI fee knowledge: ${error.message}`);
    }
  }

  private createPropertyFeeSummary(feeData: ValidatedFees): PropertyFeeSummary[] {
    return Object.entries(feeData.properties).map(([propertyId, propertyFees]) => ({
      property_id: propertyId,
      property_name: propertyFees.property_info.name,
      total_fees: this.calculateTotalFees(propertyFees),
      fee_breakdown: {
        application_fees: propertyFees.application_fees.length,
        admin_fees: propertyFees.admin_fees.length,
        pet_fees: propertyFees.pet_fees.length,
        addon_fees: propertyFees.addon_fees.length
      },
      fee_ranges: {
        application_fees: this.calculateFeeRange(propertyFees.application_fees),
        admin_fees: this.calculateFeeRange(propertyFees.admin_fees),
        pet_fees: this.calculateFeeRange(propertyFees.pet_fees),
        addon_fees: this.calculateFeeRange(propertyFees.addon_fees)
      }
    }));
  }

  private identifyCommonFees(feeData: ValidatedFees): CommonFee[] {
    const feeCounts = new Map<string, number>();
    const feeAmounts = new Map<string, number[]>();

    Object.values(feeData.properties).forEach(propertyFees => {
      [...propertyFees.application_fees, ...propertyFees.admin_fees].forEach(fee => {
        const key = fee.fee_name.toLowerCase();
        feeCounts.set(key, (feeCounts.get(key) || 0) + 1);
        
        if (!feeAmounts.has(key)) {
          feeAmounts.set(key, []);
        }
        feeAmounts.get(key).push(fee.amount);
      });
    });

    return Array.from(feeCounts.entries())
      .filter(([_, count]) => count > 1)
      .map(([name, count]) => ({
        fee_name: name,
        frequency: count,
        average_amount: this.calculateAverage(feeAmounts.get(name)),
        min_amount: Math.min(...feeAmounts.get(name)),
        max_amount: Math.max(...feeAmounts.get(name))
      }))
      .sort((a, b) => b.frequency - a.frequency);
  }
}
```

### 2. AI Fee Response Generation
```typescript
class AIFeeResponseGenerator {
  async generateFeeResponse(query: FeeQuery, feeData: CachedFeeData): Promise<FeeResponse> {
    try {
      const calculation = await feeData.calculator.calculateTotalFees({
        property_id: query.property_id,
        pets: query.pets,
        addons: query.addons,
        lease_term_months: query.lease_term_months,
        include_breakdown: true
      });

      const response = await this.aiService.generateFeeResponse({
        query: query,
        calculation: calculation,
        fee_data: feeData,
        context: {
          transparency_required: true,
          include_breakdown: true,
          show_refundable_status: true
        }
      });

      return {
        response_text: response.text,
        fee_breakdown: calculation.breakdown,
        total_costs: calculation.summary,
        disclaimers: calculation.disclaimers,
        transparency_score: this.calculateTransparencyScore(calculation),
        last_updated: feeData.last_update
      };

    } catch (error) {
      throw new Error(`Failed to generate fee response: ${error.message}`);
    }
  }

  private calculateTransparencyScore(calculation: FeeCalculation): number {
    let score = 100;
    
    // Deduct points for missing information
    if (calculation.breakdown.application_fees.length === 0) score -= 10;
    if (calculation.breakdown.admin_fees.length === 0) score -= 10;
    if (calculation.disclaimers.length === 0) score -= 5;
    
    // Bonus points for detailed breakdown
    if (calculation.breakdown.application_fees.length > 1) score += 5;
    if (calculation.breakdown.addon_fees.length > 0) score += 5;
    
    return Math.max(0, Math.min(100, score));
  }
}
```

## Scheduled Sync System

### 1. Fee Sync Scheduler
```typescript
class FeeSyncScheduler {
  private syncInterval = 60 * 60 * 1000; // 1 hour
  private lastSync = null;

  constructor(private feeSyncService: FeeSyncService) {
    this.initializeScheduler();
  }

  private initializeScheduler(): void {
    // Schedule hourly fee sync
    setInterval(async () => {
      await this.performScheduledSync();
    }, this.syncInterval);

    // Perform initial sync
    this.performScheduledSync();

    console.log('Fee sync scheduler initialized with hourly runs');
  }

  private async performScheduledSync(): Promise<void> {
    try {
      console.log('Starting scheduled fee sync...');
      
      const result = await this.feeSyncService.performSync({
        sync_type: 'incremental',
        scheduled_run: true
      });
      
      this.lastSync = new Date();
      console.log('Scheduled fee sync completed successfully');
      
    } catch (error) {
      console.error('Scheduled fee sync failed:', error);
    }
  }

  async triggerManualSync(options: FeeSyncOptions): Promise<FeeSyncResult> {
    console.log('Manual fee sync triggered');
    return await this.feeSyncService.performSync(options);
  }
}
```

### 2. Fee Sync Service
```typescript
class FeeSyncService {
  constructor(
    private feeExtractor: ResManFeeExtractor,
    private validator: FeeDataValidator,
    private processor: FeeDataProcessor,
    private aiUpdater: AIFeeKnowledgeUpdater
  ) {}

  async performSync(options: FeeSyncOptions): Promise<FeeSyncResult> {
    const startTime = Date.now();
    
    try {
      console.log('Starting ResMan fee sync...');

      // 1. Extract fee data from ResMan
      const feeData = await this.feeExtractor.extractFeeData(options.properties);
      
      // 2. Validate fee data
      const validation = this.validator.validateFeeData(feeData);
      
      if (validation.error_count > 0) {
        console.warn(`Fee sync completed with ${validation.error_count} errors`);
      }
      
      // 3. Process and cache fee data
      await this.processor.processAndCacheFeeData(validation.valid_fees);
      
      // 4. Update AI knowledge base
      await this.aiUpdater.updateFeeKnowledge(validation.valid_fees);
      
      const processingTime = Date.now() - startTime;
      
      return {
        success: true,
        sync_id: this.generateSyncId(),
        properties_processed: validation.valid_properties,
        fee_types_processed: this.countFeeTypes(validation.valid_fees),
        processing_time_ms: processingTime,
        sync_summary: {
          application_fees: this.countFeesByType(validation.valid_fees, 'application'),
          admin_fees: this.countFeesByType(validation.valid_fees, 'admin'),
          pet_fees: this.countFeesByType(validation.valid_fees, 'pet'),
          addon_fees: this.countFeesByType(validation.valid_fees, 'addon'),
          fee_updates: this.countFeeUpdates(validation.valid_fees)
        },
        errors: validation.errors,
        warnings: validation.warnings
      };
      
    } catch (error) {
      console.error('Fee sync failed:', error);
      throw error;
    }
  }
}
```

## Monitoring and Health Checks

### 1. Fee Sync Health Monitor
```typescript
class FeeSyncHealthMonitor {
  private healthMetrics = {
    last_successful_sync: null,
    consecutive_failures: 0,
    total_syncs: 0,
    successful_syncs: 0,
    average_processing_time: 0
  };

  recordSyncResult(result: FeeSyncResult): void {
    this.healthMetrics.total_syncs++;
    
    if (result.success) {
      this.healthMetrics.successful_syncs++;
      this.healthMetrics.last_successful_sync = new Date();
      this.healthMetrics.consecutive_failures = 0;
    } else {
      this.healthMetrics.consecutive_failures++;
    }
    
    // Update average processing time
    const totalTime = this.healthMetrics.average_processing_time * (this.healthMetrics.total_syncs - 1) + result.processing_time_ms;
    this.healthMetrics.average_processing_time = totalTime / this.healthMetrics.total_syncs;
  }

  getHealthStatus(): FeeHealthStatus {
    const lastSync = this.healthMetrics.last_successful_sync;
    const minutesSinceLastSync = lastSync ? 
      (Date.now() - lastSync.getTime()) / (1000 * 60) : Infinity;
    
    return {
      status: this.determineHealthStatus(minutesSinceLastSync),
      last_successful_sync: lastSync?.toISOString(),
      minutes_since_last_sync: Math.round(minutesSinceLastSync),
      consecutive_failures: this.healthMetrics.consecutive_failures,
      success_rate: this.healthMetrics.total_syncs > 0 ? 
        (this.healthMetrics.successful_syncs / this.healthMetrics.total_syncs) * 100 : 0,
      average_processing_time_ms: Math.round(this.healthMetrics.average_processing_time)
    };
  }

  private determineHealthStatus(minutesSinceLastSync: number): 'healthy' | 'degraded' | 'unhealthy' {
    if (minutesSinceLastSync <= 60) return 'healthy';
    if (minutesSinceLastSync <= 120) return 'degraded';
    return 'unhealthy';
  }
}
```

## Testing Strategy

### Unit Tests
- Fee extraction functions
- Validation logic
- Fee calculation algorithms
- AI response generation

### Integration Tests
- End-to-end fee sync flow
- ResMan API integration
- Fee calculator accuracy
- AI knowledge updates

### User Acceptance Tests
- Manual fee sync triggering
- Fee calculation accuracy
- AI fee transparency responses
- Health monitoring

## Deployment Checklist

### Pre-deployment
- [ ] ResMan API access configured
- [ ] Fee sync system deployed
- [ ] Fee validation rules configured
- [ ] AI integration ready
- [ ] Health monitoring setup

### Post-deployment
- [ ] Verify scheduled fee syncs are running
- [ ] Test manual fee sync triggering
- [ ] Validate fee calculation accuracy
- [ ] Check AI fee transparency responses
- [ ] Monitor health metrics

## Success Criteria
- [ ] 100% scheduled fee sync success rate
- [ ] <2 minute fee sync processing time
- [ ] 99%+ fee calculation accuracy
- [ ] Real-time AI fee knowledge updates
- [ ] Complete fee transparency in all interactions
- [ ] Zero hidden fees or surprises 
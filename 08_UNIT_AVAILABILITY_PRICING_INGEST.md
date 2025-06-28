# Unit Availability & Pricing Ingest: ResMan → DomIQ

## Feature Overview
Automated system that pulls unit inventory, rental rates, and concessions from ResMan three times daily (morning, afternoon, evening) to ensure AI only quotes and books tours for units that are actually available in ResMan.

## What It Delivers for On-site Teams
- **Real-time Accuracy**: AI always has current unit availability and pricing
- **No False Promises**: AI won't quote unavailable units or outdated prices
- **Automated Sync**: No manual data entry required
- **Consistent Pricing**: Same rates shown across all channels
- **Concession Tracking**: AI can offer current promotions and deals
- **Inventory Management**: Complete visibility into unit status

## Data Flow Architecture

### 1. Scheduled Ingest Flow
```
ResMan API → Data Extraction → Validation → DomIQ Update → AI Knowledge Base → Tour Booking
```

### 2. Three-Times-Daily Schedule
```
6:00 AM → Morning Ingest → Update AI for day's tours
2:00 PM → Afternoon Ingest → Mid-day availability refresh
8:00 PM → Evening Ingest → Prepare for next day
```

## Implementation Requirements

### No Database Required
This system operates with in-memory caching and direct ResMan API integration. No persistent storage needed.

### API Endpoints to Create

#### 1. Manual Ingest Trigger API
```typescript
// POST /api/resman/ingest/trigger
interface IngestTriggerRequest {
  force_refresh?: boolean; // Force immediate ingest regardless of schedule
  ingest_type?: 'full' | 'incremental'; // Full or incremental update
  properties?: string[]; // Specific properties to ingest (optional)
}

interface IngestTriggerResponse {
  success: boolean;
  ingest_id: string;
  properties_processed: number;
  units_processed: number;
  processing_time_ms: number;
  ingest_summary: {
    available_units: number;
    unavailable_units: number;
    price_updates: number;
    concession_updates: number;
  };
  errors?: string[];
}
```

#### 2. Ingest Status API
```typescript
// GET /api/resman/ingest/status
interface IngestStatusResponse {
  last_ingest: {
    timestamp: string;
    success: boolean;
    properties_processed: number;
    units_processed: number;
    processing_time_ms: number;
  };
  next_scheduled_ingest: string;
  ingest_schedule: {
    morning: string; // "06:00"
    afternoon: string; // "14:00"
    evening: string; // "20:00"
    timezone: string;
  };
  current_cache_status: {
    total_units: number;
    available_units: number;
    properties_with_availability: number;
    cache_age_minutes: number;
    is_stale: boolean; // True if cache is older than 8 hours
  };
}
```

#### 3. Unit Availability Query API
```typescript
// GET /api/resman/units/availability
interface UnitAvailabilityRequest {
  property_id?: string;
  unit_type?: string;
  min_bedrooms?: number;
  max_bedrooms?: number;
  min_price?: number;
  max_price?: number;
  available_from?: string; // Date
  available_to?: string; // Date
  include_pricing?: boolean;
  include_concessions?: boolean;
}

interface UnitAvailabilityResponse {
  units: Array<{
    unit_id: string;
    property_id: string;
    property_name: string;
    unit_number: string;
    unit_type: string;
    bedrooms: number;
    bathrooms: number;
    square_feet: number;
    floor_plan: string;
    floor: number;
    availability_status: 'available' | 'unavailable' | 'pending' | 'reserved';
    available_date: string;
    current_rent: number;
    market_rent: number;
    concessions: Array<{
      type: string; // 'free_rent', 'discount', 'amenity', 'other'
      description: string;
      value: number;
      duration_months?: number;
      valid_until?: string;
    }>;
    amenities: string[];
    last_updated: string;
  }>;
  summary: {
    total_units: number;
    available_units: number;
    price_range: {
      min: number;
      max: number;
    };
    bedroom_distribution: Record<string, number>;
    average_rent: number;
  };
}
```

#### 4. Property Summary API
```typescript
// GET /api/resman/properties/{property_id}/summary
interface PropertySummaryResponse {
  property_id: string;
  property_name: string;
  address: {
    street: string;
    city: string;
    state: string;
    zip: string;
  };
  contact_info: {
    phone: string;
    email: string;
    website: string;
  };
  availability_summary: {
    total_units: number;
    available_units: number;
    occupancy_rate: number;
    average_rent: number;
    price_range: {
      min: number;
      max: number;
    };
  };
  unit_types: Array<{
    type: string;
    bedrooms: number;
    bathrooms: number;
    available_count: number;
    starting_rent: number;
    average_rent: number;
  }>;
  current_concessions: Array<{
    type: string;
    description: string;
    value: number;
    valid_until: string;
    applicable_units: string[];
  }>;
  last_updated: string;
}
```

## ResMan Data Extraction

### 1. Unit Inventory Extraction
```typescript
class ResManUnitExtractor {
  async extractUnitInventory(propertyIds?: string[]): Promise<UnitInventory> {
    try {
      // Get all properties if none specified
      const properties = propertyIds || await this.getAllPropertyIds();
      
      const allUnits: Unit[] = [];
      
      for (const propertyId of properties) {
        const propertyUnits = await this.extractPropertyUnits(propertyId);
        allUnits.push(...propertyUnits);
      }
      
      return {
        units: allUnits,
        extraction_timestamp: new Date().toISOString(),
        total_units: allUnits.length,
        available_units: allUnits.filter(u => u.availability_status === 'available').length
      };
      
    } catch (error) {
      throw new Error(`Failed to extract unit inventory: ${error.message}`);
    }
  }

  private async extractPropertyUnits(propertyId: string): Promise<Unit[]> {
    const units = await this.resManAPI.getUnits({
      property_id: propertyId,
      include_availability: true,
      include_pricing: true,
      include_concessions: true
    });

    return units.map(unit => this.mapResManUnit(unit));
  }

  private mapResManUnit(resmanUnit: any): Unit {
    return {
      unit_id: resmanUnit.id,
      property_id: resmanUnit.property_id,
      property_name: resmanUnit.property_name,
      unit_number: resmanUnit.unit_number,
      unit_type: resmanUnit.unit_type,
      bedrooms: resmanUnit.bedrooms,
      bathrooms: resmanUnit.bathrooms,
      square_feet: resmanUnit.square_feet,
      floor_plan: resmanUnit.floor_plan,
      floor: resmanUnit.floor,
      availability_status: this.mapAvailabilityStatus(resmanUnit.status),
      available_date: resmanUnit.available_date,
      current_rent: resmanUnit.current_rent,
      market_rent: resmanUnit.market_rent,
      concessions: this.mapConcessions(resmanUnit.concessions),
      amenities: resmanUnit.amenities || [],
      last_updated: resmanUnit.last_updated
    };
  }

  private mapAvailabilityStatus(resmanStatus: string): string {
    const statusMap = {
      'Available': 'available',
      'Unavailable': 'unavailable',
      'Pending': 'pending',
      'Reserved': 'reserved',
      'Leased': 'unavailable',
      'Maintenance': 'unavailable'
    };
    
    return statusMap[resmanStatus] || 'unavailable';
  }

  private mapConcessions(resmanConcessions: any[]): Concession[] {
    return resmanConcessions.map(concession => ({
      type: concession.type,
      description: concession.description,
      value: concession.value,
      duration_months: concession.duration_months,
      valid_until: concession.valid_until
    }));
  }
}
```

### 2. Pricing and Concession Extraction
```typescript
class ResManPricingExtractor {
  async extractPricingData(propertyIds?: string[]): Promise<PricingData> {
    try {
      const properties = propertyIds || await this.getAllPropertyIds();
      
      const pricingData: PricingData = {
        properties: {},
        extraction_timestamp: new Date().toISOString()
      };
      
      for (const propertyId of properties) {
        const propertyPricing = await this.extractPropertyPricing(propertyId);
        pricingData.properties[propertyId] = propertyPricing;
      }
      
      return pricingData;
      
    } catch (error) {
      throw new Error(`Failed to extract pricing data: ${error.message}`);
    }
  }

  private async extractPropertyPricing(propertyId: string): Promise<PropertyPricing> {
    const [units, concessions, marketData] = await Promise.all([
      this.resManAPI.getUnits({ property_id: propertyId, include_pricing: true }),
      this.resManAPI.getConcessions({ property_id: propertyId }),
      this.resManAPI.getMarketData({ property_id: propertyId })
    ]);

    return {
      units: units.map(unit => ({
        unit_id: unit.id,
        current_rent: unit.current_rent,
        market_rent: unit.market_rent,
        rent_history: unit.rent_history,
        last_price_update: unit.last_price_update
      })),
      concessions: concessions.map(concession => ({
        id: concession.id,
        type: concession.type,
        description: concession.description,
        value: concession.value,
        applicable_units: concession.applicable_units,
        valid_from: concession.valid_from,
        valid_until: concession.valid_until
      })),
      market_data: {
        average_rent: marketData.average_rent,
        rent_trend: marketData.rent_trend,
        occupancy_rate: marketData.occupancy_rate,
        days_on_market: marketData.days_on_market
      }
    };
  }
}
```

## Data Validation and Processing

### 1. Data Validation Engine
```typescript
class UnitDataValidator {
  validateUnitData(units: Unit[]): ValidationResult {
    const errors: string[] = [];
    const warnings: string[] = [];
    const validUnits: Unit[] = [];

    for (const unit of units) {
      const unitValidation = this.validateUnit(unit);
      
      if (unitValidation.isValid) {
        validUnits.push(unit);
      } else {
        errors.push(`Unit ${unit.unit_id}: ${unitValidation.errors.join(', ')}`);
      }
      
      if (unitValidation.warnings.length > 0) {
        warnings.push(`Unit ${unit.unit_id}: ${unitValidation.warnings.join(', ')}`);
      }
    }

    return {
      valid_units: validUnits,
      total_units: units.length,
      valid_count: validUnits.length,
      error_count: errors.length,
      warning_count: warnings.length,
      errors,
      warnings
    };
  }

  private validateUnit(unit: Unit): UnitValidation {
    const errors: string[] = [];
    const warnings: string[] = [];

    // Required field validation
    if (!unit.unit_id) errors.push('Missing unit ID');
    if (!unit.property_id) errors.push('Missing property ID');
    if (!unit.unit_number) errors.push('Missing unit number');
    if (unit.bedrooms < 0) errors.push('Invalid bedroom count');
    if (unit.bathrooms < 0) errors.push('Invalid bathroom count');
    if (unit.current_rent < 0) errors.push('Invalid current rent');
    if (unit.market_rent < 0) errors.push('Invalid market rent');

    // Business logic validation
    if (unit.current_rent > unit.market_rent * 1.5) {
      warnings.push('Current rent significantly higher than market rent');
    }

    if (unit.availability_status === 'available' && !unit.available_date) {
      warnings.push('Available unit missing available date');
    }

    if (unit.concessions.some(c => new Date(c.valid_until) < new Date())) {
      warnings.push('Unit has expired concessions');
    }

    return {
      isValid: errors.length === 0,
      errors,
      warnings
    };
  }
}
```

### 2. Data Processing and Caching
```typescript
class UnitDataProcessor {
  private cache = new Map<string, any>();
  private lastUpdate = null;

  async processAndCacheUnitData(units: Unit[]): Promise<void> {
    try {
      // Process units into optimized format for AI queries
      const processedData = this.processUnits(units);
      
      // Update cache
      this.cache.set('units', processedData);
      this.cache.set('properties', this.groupByProperty(units));
      this.cache.set('availability', this.createAvailabilityIndex(units));
      this.cache.set('pricing', this.createPricingIndex(units));
      this.cache.set('concessions', this.createConcessionsIndex(units));
      
      this.lastUpdate = new Date();
      
      console.log(`Processed and cached ${units.length} units`);
      
    } catch (error) {
      throw new Error(`Failed to process unit data: ${error.message}`);
    }
  }

  private processUnits(units: Unit[]): ProcessedUnitData {
    return {
      by_id: new Map(units.map(u => [u.unit_id, u])),
      by_property: this.groupByProperty(units),
      by_type: this.groupByType(units),
      by_availability: this.groupByAvailability(units),
      by_price_range: this.groupByPriceRange(units),
      search_index: this.createSearchIndex(units)
    };
  }

  private createSearchIndex(units: Unit[]): SearchIndex {
    const index: SearchIndex = {
      property_names: new Set(),
      unit_types: new Set(),
      amenities: new Set(),
      price_ranges: new Map(),
      bedroom_counts: new Set()
    };

    units.forEach(unit => {
      index.property_names.add(unit.property_name);
      index.unit_types.add(unit.unit_type);
      index.bedroom_counts.add(unit.bedrooms);
      unit.amenities.forEach(amenity => index.amenities.add(amenity));
      
      const priceRange = this.getPriceRange(unit.current_rent);
      if (!index.price_ranges.has(priceRange)) {
        index.price_ranges.set(priceRange, []);
      }
      index.price_ranges.get(priceRange).push(unit.unit_id);
    });

    return index;
  }

  getCachedData(): CachedUnitData {
    return {
      units: this.cache.get('units'),
      properties: this.cache.get('properties'),
      availability: this.cache.get('availability'),
      pricing: this.cache.get('pricing'),
      concessions: this.cache.get('concessions'),
      last_update: this.lastUpdate,
      cache_age_minutes: this.lastUpdate ? 
        Math.floor((Date.now() - this.lastUpdate.getTime()) / 60000) : null
    };
  }
}
```

## Scheduled Ingest System

### 1. Scheduler Implementation
```typescript
class IngestScheduler {
  private schedules = [
    { time: '06:00', name: 'morning' },
    { time: '14:00', name: 'afternoon' },
    { time: '20:00', name: 'evening' }
  ];

  constructor(private ingestService: IngestService) {
    this.initializeScheduler();
  }

  private initializeScheduler(): void {
    // Schedule all three daily ingests
    this.schedules.forEach(schedule => {
      this.scheduleIngest(schedule.time, schedule.name);
    });

    console.log('Ingest scheduler initialized with 3 daily runs');
  }

  private scheduleIngest(time: string, name: string): void {
    const [hour, minute] = time.split(':').map(Number);
    
    cron.schedule(`${minute} ${hour} * * *`, async () => {
      console.log(`Starting ${name} ingest at ${time}`);
      
      try {
        await this.ingestService.performIngest({
          ingest_type: 'full',
          scheduled_run: name
        });
        
        console.log(`${name} ingest completed successfully`);
      } catch (error) {
        console.error(`${name} ingest failed:`, error);
      }
    });
  }

  async triggerManualIngest(options: IngestOptions): Promise<IngestResult> {
    console.log('Manual ingest triggered');
    return await this.ingestService.performIngest(options);
  }
}
```

### 2. Ingest Service
```typescript
class IngestService {
  constructor(
    private unitExtractor: ResManUnitExtractor,
    private pricingExtractor: ResManPricingExtractor,
    private validator: UnitDataValidator,
    private processor: UnitDataProcessor
  ) {}

  async performIngest(options: IngestOptions): Promise<IngestResult> {
    const startTime = Date.now();
    
    try {
      console.log('Starting ResMan data ingest...');

      // 1. Extract unit inventory
      const unitInventory = await this.unitExtractor.extractUnitInventory(options.properties);
      
      // 2. Extract pricing data
      const pricingData = await this.pricingExtractor.extractPricingData(options.properties);
      
      // 3. Validate data
      const validation = this.validator.validateUnitData(unitInventory.units);
      
      if (validation.error_count > 0) {
        console.warn(`Ingest completed with ${validation.error_count} errors`);
      }
      
      // 4. Process and cache data
      await this.processor.processAndCacheUnitData(validation.valid_units);
      
      // 5. Update AI knowledge base
      await this.updateAIKnowledgeBase(validation.valid_units, pricingData);
      
      const processingTime = Date.now() - startTime;
      
      return {
        success: true,
        ingest_id: this.generateIngestId(),
        properties_processed: this.countProperties(validation.valid_units),
        units_processed: validation.valid_units.length,
        processing_time_ms: processingTime,
        ingest_summary: {
          available_units: validation.valid_units.filter(u => u.availability_status === 'available').length,
          unavailable_units: validation.valid_units.filter(u => u.availability_status !== 'available').length,
          price_updates: this.countPriceUpdates(pricingData),
          concession_updates: this.countConcessionUpdates(pricingData)
        },
        errors: validation.errors,
        warnings: validation.warnings
      };
      
    } catch (error) {
      console.error('Ingest failed:', error);
      throw error;
    }
  }

  private async updateAIKnowledgeBase(units: Unit[], pricingData: PricingData): Promise<void> {
    // Update AI system with current unit availability and pricing
    await this.aiService.updateUnitKnowledge({
      units: units,
      pricing: pricingData,
      update_timestamp: new Date().toISOString()
    });
  }
}
```

## AI Integration

### 1. AI Knowledge Base Update
```typescript
class AIKnowledgeUpdater {
  async updateUnitKnowledge(data: UnitKnowledgeData): Promise<void> {
    try {
      // Update AI's understanding of current availability
      await this.aiService.updateContext({
        type: 'unit_availability',
        data: {
          available_units: data.units.filter(u => u.availability_status === 'available'),
          total_units: data.units.length,
          properties: this.createPropertySummary(data.units),
          pricing: this.createPricingSummary(data.pricing),
          concessions: this.createConcessionsSummary(data.pricing)
        },
        timestamp: data.update_timestamp
      });

      // Update tour booking constraints
      await this.aiService.updateConstraints({
        type: 'tour_booking',
        constraints: {
          only_available_units: true,
          current_pricing_only: true,
          include_concessions: true,
          max_advance_booking_days: 30
        }
      });

      console.log('AI knowledge base updated with current unit data');
      
    } catch (error) {
      throw new Error(`Failed to update AI knowledge base: ${error.message}`);
    }
  }

  private createPropertySummary(units: Unit[]): PropertySummary[] {
    const propertyMap = new Map<string, PropertySummary>();
    
    units.forEach(unit => {
      if (!propertyMap.has(unit.property_id)) {
        propertyMap.set(unit.property_id, {
          property_id: unit.property_id,
          property_name: unit.property_name,
          total_units: 0,
          available_units: 0,
          unit_types: new Set(),
          price_range: { min: Infinity, max: 0 }
        });
      }
      
      const summary = propertyMap.get(unit.property_id);
      summary.total_units++;
      if (unit.availability_status === 'available') {
        summary.available_units++;
      }
      summary.unit_types.add(unit.unit_type);
      summary.price_range.min = Math.min(summary.price_range.min, unit.current_rent);
      summary.price_range.max = Math.max(summary.price_range.max, unit.current_rent);
    });
    
    return Array.from(propertyMap.values());
  }
}
```

### 2. Tour Booking Validation
```typescript
class TourBookingValidator {
  async validateTourRequest(request: TourRequest): Promise<ValidationResult> {
    const cachedData = this.unitProcessor.getCachedData();
    
    // Check if unit is available
    const unit = cachedData.units.by_id.get(request.unit_id);
    if (!unit) {
      return {
        valid: false,
        error: 'Unit not found'
      };
    }
    
    if (unit.availability_status !== 'available') {
      return {
        valid: false,
        error: `Unit ${unit.unit_number} is not available (status: ${unit.availability_status})`
      };
    }
    
    // Check if available date is valid
    if (unit.available_date && new Date(unit.available_date) > new Date(request.tour_date)) {
      return {
        valid: false,
        error: `Unit ${unit.unit_number} is not available until ${unit.available_date}`
      };
    }
    
    // Check if pricing is current
    const cacheAge = cachedData.cache_age_minutes;
    if (cacheAge > 480) { // 8 hours
      return {
        valid: false,
        error: 'Unit availability data is stale. Please try again in a few minutes.'
      };
    }
    
    return {
      valid: true,
      unit: unit,
      pricing: {
        current_rent: unit.current_rent,
        market_rent: unit.market_rent,
        concessions: unit.concessions
      }
    };
  }
}
```

## Monitoring and Health Checks

### 1. Ingest Health Monitoring
```typescript
class IngestHealthMonitor {
  private healthMetrics = {
    last_successful_ingest: null,
    consecutive_failures: 0,
    total_ingests: 0,
    successful_ingests: 0,
    average_processing_time: 0
  };

  recordIngestResult(result: IngestResult): void {
    this.healthMetrics.total_ingests++;
    
    if (result.success) {
      this.healthMetrics.successful_ingests++;
      this.healthMetrics.last_successful_ingest = new Date();
      this.healthMetrics.consecutive_failures = 0;
    } else {
      this.healthMetrics.consecutive_failures++;
    }
    
    // Update average processing time
    const totalTime = this.healthMetrics.average_processing_time * (this.healthMetrics.total_ingests - 1) + result.processing_time_ms;
    this.healthMetrics.average_processing_time = totalTime / this.healthMetrics.total_ingests;
  }

  getHealthStatus(): HealthStatus {
    const lastIngest = this.healthMetrics.last_successful_ingest;
    const hoursSinceLastIngest = lastIngest ? 
      (Date.now() - lastIngest.getTime()) / (1000 * 60 * 60) : Infinity;
    
    return {
      status: this.determineHealthStatus(hoursSinceLastIngest),
      last_successful_ingest: lastIngest?.toISOString(),
      hours_since_last_ingest: Math.round(hoursSinceLastIngest),
      consecutive_failures: this.healthMetrics.consecutive_failures,
      success_rate: this.healthMetrics.total_ingests > 0 ? 
        (this.healthMetrics.successful_ingests / this.healthMetrics.total_ingests) * 100 : 0,
      average_processing_time_ms: Math.round(this.healthMetrics.average_processing_time)
    };
  }

  private determineHealthStatus(hoursSinceLastIngest: number): 'healthy' | 'degraded' | 'unhealthy' {
    if (hoursSinceLastIngest <= 8) return 'healthy';
    if (hoursSinceLastIngest <= 24) return 'degraded';
    return 'unhealthy';
  }
}
```

## Testing Strategy

### Unit Tests
- Data extraction functions
- Validation logic
- Processing and caching
- AI knowledge updates

### Integration Tests
- End-to-end ingest flow
- ResMan API integration
- Scheduled ingest execution
- Cache management

### User Acceptance Tests
- Manual ingest triggering
- Data accuracy verification
- AI tour booking validation
- Health monitoring

## Deployment Checklist

### Pre-deployment
- [ ] ResMan API access configured
- [ ] Scheduled ingest system deployed
- [ ] Data validation rules configured
- [ ] AI integration ready
- [ ] Health monitoring setup

### Post-deployment
- [ ] Verify scheduled ingests are running
- [ ] Test manual ingest triggering
- [ ] Validate data accuracy
- [ ] Check AI knowledge updates
- [ ] Monitor health metrics

## Success Criteria
- [ ] 100% scheduled ingest success rate
- [ ] <5 minute ingest processing time
- [ ] 99%+ data validation accuracy
- [ ] Real-time AI knowledge updates
- [ ] Zero stale data in tour bookings
- [ ] Three-times-daily schedule maintained 
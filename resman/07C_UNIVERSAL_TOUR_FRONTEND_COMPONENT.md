# Universal Tour Frontend Component: Multi-Source → ResMan

## Frontend Implementation for Universal Tour Integration

### UI Components

#### 1. Universal Tour Submission Form
```typescript
// components/tours/UniversalTourForm.tsx
interface UniversalTourFormProps {
  source: string;
  onSubmit?: (tourData: UniversalTourSubmissionRequest) => Promise<void>;
  onSuccess?: (response: UniversalTourSubmissionResponse) => void;
  onError?: (error: string) => void;
}

const UniversalTourForm: React.FC<UniversalTourFormProps> = ({
  source,
  onSubmit,
  onSuccess,
  onError
}) => {
  const [formData, setFormData] = useState<any>({});
  const [submitting, setSubmitting] = useState(false);
  const [validationErrors, setValidationErrors] = useState<string[]>([]);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setSubmitting(true);
    setValidationErrors([]);

    try {
      const tourSubmission: UniversalTourSubmissionRequest = {
        source,
        source_id: generateSourceId(),
        tour_data: formData,
        metadata: {
          submitted_by: getCurrentUser(),
          submission_method: 'web_form',
          source_url: window.location.href,
          user_agent: navigator.userAgent,
          ip_address: await getClientIP()
        }
      };

      const response = await fetch('/api/universal/tours/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(tourSubmission)
      });

      const result = await response.json();

      if (result.success) {
        onSuccess?.(result);
        setFormData({});
      } else {
        setValidationErrors(result.validation_errors || [result.error]);
        onError?.(result.error);
      }
    } catch (error) {
      setValidationErrors([error.message]);
      onError?.(error.message);
    } finally {
      setSubmitting(false);
    }
  };

  return (
    <div className="universal-tour-form">
      <form onSubmit={handleSubmit} className="space-y-6">
        {/* Prospect Information */}
        <div className="prospect-section">
          <h3 className="text-lg font-medium text-gray-900 mb-4">Prospect Information</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Name *
              </label>
              <input
                type="text"
                required
                value={formData.prospect_name || ''}
                onChange={(e) => setFormData({ ...formData, prospect_name: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Email
              </label>
              <input
                type="email"
                value={formData.prospect_email || ''}
                onChange={(e) => setFormData({ ...formData, prospect_email: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Phone
              </label>
              <input
                type="tel"
                value={formData.prospect_phone || ''}
                onChange={(e) => setFormData({ ...formData, prospect_phone: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
          </div>
        </div>

        {/* Tour Information */}
        <div className="tour-section">
          <h3 className="text-lg font-medium text-gray-900 mb-4">Tour Details</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Property *
              </label>
              <select
                required
                value={formData.property_name || ''}
                onChange={(e) => setFormData({ ...formData, property_name: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              >
                <option value="">Select Property</option>
                <option value="property_1">Sunset Apartments</option>
                <option value="property_2">Downtown Lofts</option>
                <option value="property_3">Garden Villas</option>
              </select>
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Tour Type
              </label>
              <select
                value={formData.tour_type || 'agent_led'}
                onChange={(e) => setFormData({ ...formData, tour_type: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              >
                <option value="agent_led">Agent Led</option>
                <option value="self_guided">Self Guided</option>
                <option value="virtual">Virtual Tour</option>
              </select>
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Preferred Date *
              </label>
              <input
                type="date"
                required
                value={formData.preferred_date || ''}
                onChange={(e) => setFormData({ ...formData, preferred_date: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Preferred Time *
              </label>
              <input
                type="time"
                required
                value={formData.preferred_time || ''}
                onChange={(e) => setFormData({ ...formData, preferred_time: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
          </div>
        </div>

        {/* Additional Information */}
        <div className="additional-section">
          <h3 className="text-lg font-medium text-gray-900 mb-4">Additional Information</h3>
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Group Size
              </label>
              <input
                type="number"
                min="1"
                value={formData.group_size || ''}
                onChange={(e) => setFormData({ ...formData, group_size: parseInt(e.target.value) })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              />
            </div>
            <div>
              <label className="block text-sm font-medium text-gray-700 mb-1">
                Special Requests
              </label>
              <textarea
                value={formData.special_requests || ''}
                onChange={(e) => setFormData({ ...formData, special_requests: e.target.value })}
                rows={3}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                placeholder="Any special requirements or requests..."
              />
            </div>
          </div>
        </div>

        {/* Validation Errors */}
        {validationErrors.length > 0 && (
          <div className="validation-errors">
            <div className="bg-red-50 border border-red-200 rounded-md p-4">
              <h4 className="text-sm font-medium text-red-800 mb-2">Please fix the following errors:</h4>
              <ul className="text-sm text-red-700 space-y-1">
                {validationErrors.map((error, index) => (
                  <li key={index}>• {error}</li>
                ))}
              </ul>
            </div>
          </div>
        )}

        {/* Submit Button */}
        <div className="submit-section">
          <button
            type="submit"
            disabled={submitting}
            className="w-full bg-blue-600 text-white py-3 px-4 rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 disabled:opacity-50 disabled:cursor-not-allowed"
          >
            {submitting ? (
              <div className="flex items-center justify-center">
                <LoadingSpinner size="sm" className="mr-2" />
                Submitting Tour Request...
              </div>
            ) : (
              'Submit Tour Request'
            )}
          </button>
        </div>
      </form>
    </div>
  );
};
```

#### 2. Tour Submission Success Modal
```typescript
// components/tours/TourSuccessModal.tsx
interface TourSuccessModalProps {
  isOpen: boolean;
  onClose: () => void;
  tourResponse: UniversalTourSubmissionResponse;
}

const TourSuccessModal: React.FC<TourSuccessModalProps> = ({
  isOpen,
  onClose,
  tourResponse
}) => {
  if (!isOpen) return null;

  return (
    <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-6 max-w-md w-full mx-4">
        <div className="text-center">
          <div className="mx-auto flex items-center justify-center h-12 w-12 rounded-full bg-green-100 mb-4">
            <CheckIcon className="h-6 w-6 text-green-600" />
          </div>
          
          <h3 className="text-lg font-medium text-gray-900 mb-2">
            Tour Request Submitted Successfully!
          </h3>
          
          <p className="text-sm text-gray-600 mb-4">
            Your tour request has been processed and sent to our team.
          </p>
          
          <div className="bg-gray-50 rounded-md p-4 mb-4">
            <div className="text-sm text-gray-600">
              <div className="flex justify-between mb-1">
                <span>ResMan Tour ID:</span>
                <span className="font-mono text-gray-900">{tourResponse.resman_tour_id}</span>
              </div>
              <div className="flex justify-between mb-1">
                <span>Processing Time:</span>
                <span className="text-gray-900">{tourResponse.processing_time_ms}ms</span>
              </div>
            </div>
          </div>
          
          {tourResponse.warnings && tourResponse.warnings.length > 0 && (
            <div className="bg-yellow-50 border border-yellow-200 rounded-md p-3 mb-4">
              <h4 className="text-sm font-medium text-yellow-800 mb-1">Warnings:</h4>
              <ul className="text-sm text-yellow-700 space-y-1">
                {tourResponse.warnings.map((warning, index) => (
                  <li key={index}>• {warning}</li>
                ))}
              </ul>
            </div>
          )}
          
          <button
            onClick={onClose}
            className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500"
          >
            Close
          </button>
        </div>
      </div>
    </div>
  );
};
```

#### 3. Tour Integration Dashboard
```typescript
// components/tours/TourIntegrationDashboard.tsx
interface TourIntegrationDashboardProps {
  source: string;
}

const TourIntegrationDashboard: React.FC<TourIntegrationDashboardProps> = ({ source }) => {
  const [stats, setStats] = useState<TourStats | null>(null);
  const [recentSubmissions, setRecentSubmissions] = useState<TourSubmission[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchDashboardData();
  }, [source]);

  const fetchDashboardData = async () => {
    setLoading(true);
    try {
      // Fetch health status
      const healthResponse = await fetch('/api/universal/tours/health');
      const healthData = await healthResponse.json();
      
      // Fetch recent submissions (if logging is enabled)
      const submissionsResponse = await fetch(`/api/universal/tours/recent?source=${source}`);
      const submissionsData = await submissionsResponse.json();
      
      setStats(healthData);
      setRecentSubmissions(submissionsData.submissions || []);
    } catch (error) {
      console.error('Error fetching dashboard data:', error);
    } finally {
      setLoading(false);
    }
  };

  if (loading) {
    return <LoadingSpinner />;
  }

  return (
    <div className="tour-integration-dashboard">
      <div className="mb-6">
        <h2 className="text-2xl font-bold text-gray-900 mb-2">
          Tour Integration Dashboard
        </h2>
        <p className="text-gray-600">
          Source: {source} | Status: {stats?.status}
        </p>
      </div>

      {/* Health Status Cards */}
      <div className="grid grid-cols-1 md:grid-cols-4 gap-4 mb-6">
        <div className="bg-white p-4 rounded-lg shadow">
          <div className="flex items-center">
            <div className={`w-3 h-3 rounded-full mr-2 ${
              stats?.resman_connection === 'connected' ? 'bg-green-500' : 'bg-red-500'
            }`} />
            <span className="text-sm font-medium text-gray-900">ResMan Connection</span>
          </div>
          <p className="text-lg font-semibold text-gray-900 mt-1">
            {stats?.resman_connection}
          </p>
        </div>

        <div className="bg-white p-4 rounded-lg shadow">
          <span className="text-sm font-medium text-gray-900">Total Requests</span>
          <p className="text-lg font-semibold text-gray-900 mt-1">
            {stats?.total_requests_processed || 0}
          </p>
        </div>

        <div className="bg-white p-4 rounded-lg shadow">
          <span className="text-sm font-medium text-gray-900">Success Rate</span>
          <p className="text-lg font-semibold text-gray-900 mt-1">
            {stats?.success_rate_percent || 0}%
          </p>
        </div>

        <div className="bg-white p-4 rounded-lg shadow">
          <span className="text-sm font-medium text-gray-900">Avg Processing Time</span>
          <p className="text-lg font-semibold text-gray-900 mt-1">
            {stats?.average_processing_time_ms || 0}ms
          </p>
        </div>
      </div>

      {/* Recent Submissions */}
      {recentSubmissions.length > 0 && (
        <div className="bg-white rounded-lg shadow">
          <div className="px-6 py-4 border-b border-gray-200">
            <h3 className="text-lg font-medium text-gray-900">Recent Submissions</h3>
          </div>
          <div className="overflow-x-auto">
            <table className="min-w-full divide-y divide-gray-200">
              <thead className="bg-gray-50">
                <tr>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Time
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Status
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    Processing Time
                  </th>
                  <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                    ResMan ID
                  </th>
                </tr>
              </thead>
              <tbody className="bg-white divide-y divide-gray-200">
                {recentSubmissions.map((submission) => (
                  <tr key={submission.id}>
                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                      {formatTimestamp(submission.created_at)}
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap">
                      <span className={`inline-flex px-2 py-1 text-xs font-semibold rounded-full ${
                        submission.processing_status === 'success' 
                          ? 'bg-green-100 text-green-800'
                          : submission.processing_status === 'failed'
                          ? 'bg-red-100 text-red-800'
                          : 'bg-yellow-100 text-yellow-800'
                      }`}>
                        {submission.processing_status}
                      </span>
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900">
                      {submission.processing_time_ms}ms
                    </td>
                    <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-900 font-mono">
                      {submission.resman_tour_id || '-'}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        </div>
      )}
    </div>
  );
};
```

#### 4. Source Configuration Manager
```typescript
// components/tours/SourceConfigurationManager.tsx
interface SourceConfigurationManagerProps {
  onConfigurationUpdate?: (config: SourceConfig) => void;
}

const SourceConfigurationManager: React.FC<SourceConfigurationManagerProps> = ({
  onConfigurationUpdate
}) => {
  const [configurations, setConfigurations] = useState<SourceConfig[]>([]);
  const [selectedConfig, setSelectedConfig] = useState<SourceConfig | null>(null);
  const [editing, setEditing] = useState(false);

  useEffect(() => {
    fetchConfigurations();
  }, []);

  const fetchConfigurations = async () => {
    try {
      const response = await fetch('/api/universal/tours/configurations');
      const data = await response.json();
      setConfigurations(data.configurations);
    } catch (error) {
      console.error('Error fetching configurations:', error);
    }
  };

  const handleSaveConfiguration = async (config: SourceConfig) => {
    try {
      const response = await fetch('/api/universal/tours/configurations', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(config)
      });

      if (response.ok) {
        await fetchConfigurations();
        onConfigurationUpdate?.(config);
        setEditing(false);
      }
    } catch (error) {
      console.error('Error saving configuration:', error);
    }
  };

  return (
    <div className="source-configuration-manager">
      <div className="mb-6">
        <h2 className="text-2xl font-bold text-gray-900 mb-2">
          Source Configurations
        </h2>
        <p className="text-gray-600">
          Manage data mapping and validation rules for different tour sources
        </p>
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        {/* Configuration List */}
        <div className="bg-white rounded-lg shadow">
          <div className="px-6 py-4 border-b border-gray-200">
            <h3 className="text-lg font-medium text-gray-900">Available Sources</h3>
          </div>
          <div className="divide-y divide-gray-200">
            {configurations.map((config) => (
              <div
                key={config.source_name}
                className={`px-6 py-4 cursor-pointer hover:bg-gray-50 ${
                  selectedConfig?.source_name === config.source_name ? 'bg-blue-50' : ''
                }`}
                onClick={() => setSelectedConfig(config)}
              >
                <div className="flex items-center justify-between">
                  <div>
                    <h4 className="text-sm font-medium text-gray-900">
                      {config.source_name}
                    </h4>
                    <p className="text-sm text-gray-500">
                      Type: {config.source_type}
                    </p>
                  </div>
                  <div className="flex items-center space-x-2">
                    <span className={`inline-flex px-2 py-1 text-xs font-semibold rounded-full ${
                      config.is_active ? 'bg-green-100 text-green-800' : 'bg-gray-100 text-gray-800'
                    }`}>
                      {config.is_active ? 'Active' : 'Inactive'}
                    </span>
                  </div>
                </div>
              </div>
            ))}
          </div>
        </div>

        {/* Configuration Editor */}
        {selectedConfig && (
          <div className="bg-white rounded-lg shadow">
            <div className="px-6 py-4 border-b border-gray-200">
              <div className="flex items-center justify-between">
                <h3 className="text-lg font-medium text-gray-900">
                  Edit Configuration: {selectedConfig.source_name}
                </h3>
                <button
                  onClick={() => setEditing(!editing)}
                  className="text-sm text-blue-600 hover:text-blue-800"
                >
                  {editing ? 'Cancel' : 'Edit'}
                </button>
              </div>
            </div>
            <div className="p-6">
              {editing ? (
                <ConfigurationEditor
                  config={selectedConfig}
                  onSave={handleSaveConfiguration}
                  onCancel={() => setEditing(false)}
                />
              ) : (
                <ConfigurationViewer config={selectedConfig} />
              )}
            </div>
          </div>
        )}
      </div>
    </div>
  );
};
```

### State Management

#### 1. Tour Submission Store
```typescript
// stores/tourSubmissionStore.ts
interface TourSubmissionState {
  submissions: TourSubmission[];
  currentSubmission: TourSubmission | null;
  loading: boolean;
  error: string | null;
  
  // Actions
  submitTour: (tourData: UniversalTourSubmissionRequest) => Promise<void>;
  clearError: () => void;
  resetSubmission: () => void;
}

export const useTourSubmissionStore = create<TourSubmissionState>((set, get) => ({
  submissions: [],
  currentSubmission: null,
  loading: false,
  error: null,
  
  submitTour: async (tourData) => {
    set({ loading: true, error: null });
    try {
      const response = await fetch('/api/universal/tours/submit', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(tourData)
      });
      
      const result = await response.json();
      
      if (result.success) {
        const submission: TourSubmission = {
          id: generateId(),
          ...tourData,
          response: result,
          created_at: new Date().toISOString()
        };
        
        set({
          currentSubmission: submission,
          submissions: [submission, ...get().submissions],
          loading: false
        });
      } else {
        set({
          error: result.error || 'Tour submission failed',
          loading: false
        });
      }
    } catch (error) {
      set({
        error: error.message,
        loading: false
      });
    }
  },
  
  clearError: () => set({ error: null }),
  resetSubmission: () => set({ currentSubmission: null })
}));
```

### Integration Examples

#### 1. Website Form Integration
```typescript
// pages/tours/website-form.tsx
const WebsiteTourFormPage: React.FC = () => {
  const { submitTour, currentSubmission, loading, error } = useTourSubmissionStore();
  const [showSuccessModal, setShowSuccessModal] = useState(false);

  const handleFormSubmit = async (formData: any) => {
    await submitTour({
      source: 'website',
      tour_data: formData,
      metadata: {
        submitted_by: 'website_user',
        submission_method: 'website_form',
        source_url: window.location.href
      }
    });
  };

  useEffect(() => {
    if (currentSubmission?.response?.success) {
      setShowSuccessModal(true);
    }
  }, [currentSubmission]);

  return (
    <div className="website-tour-form-page">
      <div className="max-w-2xl mx-auto py-8">
        <div className="mb-8">
          <h1 className="text-3xl font-bold text-gray-900 mb-2">
            Schedule a Tour
          </h1>
          <p className="text-gray-600">
            Fill out the form below to schedule your property tour.
          </p>
        </div>

        <UniversalTourForm
          source="website"
          onSubmit={handleFormSubmit}
          onSuccess={() => setShowSuccessModal(true)}
          onError={(error) => console.error('Form error:', error)}
        />

        {error && (
          <div className="mt-4 bg-red-50 border border-red-200 rounded-md p-4">
            <p className="text-sm text-red-700">{error}</p>
          </div>
        )}

        {currentSubmission?.response && (
          <TourSuccessModal
            isOpen={showSuccessModal}
            onClose={() => setShowSuccessModal(false)}
            tourResponse={currentSubmission.response}
          />
        )}
      </div>
    </div>
  );
};
```

#### 2. Admin Dashboard Integration
```typescript
// pages/admin/tour-integration.tsx
const TourIntegrationAdminPage: React.FC = () => {
  const [activeTab, setActiveTab] = useState<'dashboard' | 'configurations'>('dashboard');

  return (
    <div className="tour-integration-admin-page">
      <div className="max-w-7xl mx-auto py-8">
        <div className="mb-8">
          <h1 className="text-3xl font-bold text-gray-900 mb-2">
            Tour Integration Management
          </h1>
          <p className="text-gray-600">
            Monitor and configure the universal tour integration system
          </p>
        </div>

        {/* Tab Navigation */}
        <div className="border-b border-gray-200 mb-8">
          <nav className="-mb-px flex space-x-8">
            <button
              onClick={() => setActiveTab('dashboard')}
              className={`py-2 px-1 border-b-2 font-medium text-sm ${
                activeTab === 'dashboard'
                  ? 'border-blue-500 text-blue-600'
                  : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'
              }`}
            >
              Dashboard
            </button>
            <button
              onClick={() => setActiveTab('configurations')}
              className={`py-2 px-1 border-b-2 font-medium text-sm ${
                activeTab === 'configurations'
                  ? 'border-blue-500 text-blue-600'
                  : 'border-transparent text-gray-500 hover:text-gray-700 hover:border-gray-300'
              }`}
            >
              Configurations
            </button>
          </nav>
        </div>

        {/* Tab Content */}
        {activeTab === 'dashboard' && (
          <TourIntegrationDashboard source="all" />
        )}
        
        {activeTab === 'configurations' && (
          <SourceConfigurationManager />
        )}
      </div>
    </div>
  );
};
```

### Error Handling and User Feedback

#### 1. Error Boundary for Tour Components
```typescript
// components/tours/TourErrorBoundary.tsx
class TourErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  constructor(props: { children: React.ReactNode }) {
    super(props);
    this.state = { hasError: false, error: null };
  }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('Tour component error:', error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="tour-error">
          <div className="text-center py-12">
            <h3 className="text-lg font-medium text-gray-900 mb-2">
              Tour System Error
            </h3>
            <p className="text-gray-600 mb-4">
              There was an error with the tour booking system.
            </p>
            <button
              onClick={() => this.setState({ hasError: false, error: null })}
              className="bg-blue-600 text-white px-4 py-2 rounded-md hover:bg-blue-700"
            >
              Try Again
            </button>
          </div>
        </div>
      );
    }

    return this.props.children;
  }
}
```

#### 2. Loading States and Feedback
```typescript
// components/tours/TourLoadingStates.tsx
const TourLoadingSpinner: React.FC<{ message?: string }> = ({ message = 'Processing tour request...' }) => (
  <div className="flex items-center justify-center py-8">
    <div className="text-center">
      <LoadingSpinner size="lg" className="mx-auto mb-4" />
      <p className="text-gray-600">{message}</p>
    </div>
  </div>
);

const TourProcessingIndicator: React.FC<{ step: string }> = ({ step }) => (
  <div className="bg-blue-50 border border-blue-200 rounded-md p-4 mb-4">
    <div className="flex items-center">
      <LoadingSpinner size="sm" className="mr-3" />
      <div>
        <p className="text-sm font-medium text-blue-800">Processing Tour Request</p>
        <p className="text-sm text-blue-600">Current step: {step}</p>
      </div>
    </div>
  </div>
);
```

This frontend implementation provides a comprehensive UI for the Universal Tour Integration, including submission forms, monitoring dashboards, configuration management, and user-friendly error handling. The system is designed to be lightweight and stateless, with all processing happening on the backend and immediate forwarding to ResMan. 
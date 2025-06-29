# Multi-Channel Communications Frontend Component: DomIQ ↔ ResMan

## Frontend Implementation for Multi-Channel Communications Logging

### UI Components

#### 1. Communication History Component
```typescript
// components/communications/CommunicationHistory.tsx
interface CommunicationHistoryProps {
  prospectId: string;
  dateRange?: DateRange;
  communicationTypes?: string[];
  onCommunicationSelect?: (communication: Communication) => void;
}

const CommunicationHistory: React.FC<CommunicationHistoryProps> = ({
  prospectId,
  dateRange,
  communicationTypes,
  onCommunicationSelect
}) => {
  const [communications, setCommunications] = useState<Communication[]>([]);
  const [loading, setLoading] = useState(false);
  const [syncStatus, setSyncStatus] = useState<SyncStatus | null>(null);

  const fetchCommunications = async () => {
    setLoading(true);
    try {
      const response = await fetch(`/api/resman/communications/${prospectId}/history`, {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ dateRange, communicationTypes, include_sync_status: true })
      });
      
      const data = await response.json();
      setCommunications(data.communications);
      setSyncStatus(data.sync_status);
    } catch (error) {
      console.error('Error fetching communications:', error);
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    fetchCommunications();
  }, [prospectId, dateRange, communicationTypes]);

  return (
    <div className="communication-history">
      <div className="header">
        <h3>Communication History</h3>
        <SyncStatusIndicator status={syncStatus} />
      </div>
      
      {loading ? (
        <LoadingSpinner />
      ) : (
        <div className="communications-list">
          {communications.map((communication) => (
            <CommunicationCard
              key={communication.id}
              communication={communication}
              onClick={() => onCommunicationSelect?.(communication)}
            />
          ))}
        </div>
      )}
    </div>
  );
};
```

#### 2. Communication Card Component
```typescript
// components/communications/CommunicationCard.tsx
interface CommunicationCardProps {
  communication: Communication;
  onClick?: () => void;
  showSyncStatus?: boolean;
}

const CommunicationCard: React.FC<CommunicationCardProps> = ({
  communication,
  onClick,
  showSyncStatus = true
}) => {
  const getCommunicationIcon = (type: string) => {
    switch (type) {
      case 'chat': return <ChatIcon />;
      case 'email': return <EmailIcon />;
      case 'sms': return <SMSIcon />;
      case 'voice': return <PhoneIcon />;
      default: return <MessageIcon />;
    }
  };

  const getSentimentColor = (score: number) => {
    if (score > 0.3) return 'text-green-600';
    if (score < -0.3) return 'text-red-600';
    return 'text-gray-600';
  };

  return (
    <div 
      className="communication-card cursor-pointer hover:bg-gray-50 p-4 border rounded-lg"
      onClick={onClick}
    >
      <div className="flex items-start space-x-3">
        <div className="flex-shrink-0">
          {getCommunicationIcon(communication.type)}
        </div>
        
        <div className="flex-1 min-w-0">
          <div className="flex items-center justify-between">
            <div className="flex items-center space-x-2">
              <span className="text-sm font-medium text-gray-900">
                {communication.sender_type === 'ai' ? 'AI Assistant' : 'Prospect'}
              </span>
              <span className="text-xs text-gray-500">
                {formatTimestamp(communication.timestamp)}
              </span>
            </div>
            
            {showSyncStatus && communication.sync_status && (
              <SyncStatusBadge status={communication.sync_status.status} />
            )}
          </div>
          
          <div className="mt-2">
            <p className="text-sm text-gray-900 line-clamp-2">
              {communication.content_summary || communication.content}
            </p>
          </div>
          
          <div className="mt-2 flex items-center space-x-4">
            {communication.sentiment_score !== undefined && (
              <div className={`text-xs ${getSentimentColor(communication.sentiment_score)}`}>
                Sentiment: {communication.sentiment_score.toFixed(2)}
              </div>
            )}
            
            {communication.key_topics && communication.key_topics.length > 0 && (
              <div className="flex items-center space-x-1">
                <span className="text-xs text-gray-500">Topics:</span>
                {communication.key_topics.slice(0, 2).map((topic, index) => (
                  <span key={index} className="text-xs bg-blue-100 text-blue-800 px-1 rounded">
                    {topic}
                  </span>
                ))}
              </div>
            )}
          </div>
          
          {communication.compliance_flags && communication.compliance_flags.length > 0 && (
            <div className="mt-2">
              <ComplianceFlagsIndicator flags={communication.compliance_flags} />
            </div>
          )}
        </div>
      </div>
    </div>
  );
};
```

#### 3. Communication Search Component
```typescript
// components/communications/CommunicationSearch.tsx
interface CommunicationSearchProps {
  prospectId?: string;
  onSearchResults?: (results: CommunicationSearchResult[]) => void;
  onSearchMetadata?: (metadata: SearchMetadata) => void;
}

const CommunicationSearch: React.FC<CommunicationSearchProps> = ({
  prospectId,
  onSearchResults,
  onSearchMetadata
}) => {
  const [searchQuery, setSearchQuery] = useState('');
  const [filters, setFilters] = useState<SearchFilters>({
    dateFrom: null,
    dateTo: null,
    communicationTypes: [],
    sentimentFilter: null,
    complianceFlags: [],
    syncStatus: []
  });
  const [searching, setSearching] = useState(false);

  const handleSearch = async () => {
    if (!searchQuery.trim()) return;
    
    setSearching(true);
    try {
      const response = await fetch('/api/resman/communications/search', {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          prospect_id: prospectId,
          search_query: searchQuery,
          ...filters
        })
      });
      
      const data = await response.json();
      onSearchResults?.(data.results);
      onSearchMetadata?.(data.search_metadata);
    } catch (error) {
      console.error('Error searching communications:', error);
    } finally {
      setSearching(false);
    }
  };

  return (
    <div className="communication-search">
      <div className="search-input-container">
        <input
          type="text"
          value={searchQuery}
          onChange={(e) => setSearchQuery(e.target.value)}
          placeholder="Search communications..."
          className="w-full px-4 py-2 border rounded-lg"
          onKeyPress={(e) => e.key === 'Enter' && handleSearch()}
        />
        <button
          onClick={handleSearch}
          disabled={searching}
          className="ml-2 px-4 py-2 bg-blue-600 text-white rounded-lg hover:bg-blue-700 disabled:opacity-50"
        >
          {searching ? <LoadingSpinner size="sm" /> : 'Search'}
        </button>
      </div>
      
      <SearchFilters
        filters={filters}
        onFiltersChange={setFilters}
      />
    </div>
  );
};
```

#### 4. Search Filters Component
```typescript
// components/communications/SearchFilters.tsx
interface SearchFiltersProps {
  filters: SearchFilters;
  onFiltersChange: (filters: SearchFilters) => void;
}

const SearchFilters: React.FC<SearchFiltersProps> = ({ filters, onFiltersChange }) => {
  const communicationTypes = ['chat', 'email', 'sms', 'voice'];
  const sentimentOptions = ['positive', 'negative', 'neutral'];
  const complianceFlagTypes = ['gdpr_consent', 'opt_out', 'sensitive_data', 'regulatory'];
  const syncStatusOptions = ['pending', 'success', 'failed', 'conflict'];

  return (
    <div className="search-filters mt-4 p-4 bg-gray-50 rounded-lg">
      <h4 className="text-sm font-medium text-gray-900 mb-3">Filters</h4>
      
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        {/* Date Range */}
        <div>
          <label className="block text-xs font-medium text-gray-700 mb-1">
            Date Range
          </label>
          <div className="flex space-x-2">
            <input
              type="date"
              value={filters.dateFrom || ''}
              onChange={(e) => onFiltersChange({
                ...filters,
                dateFrom: e.target.value || null
              })}
              className="text-xs px-2 py-1 border rounded"
            />
            <input
              type="date"
              value={filters.dateTo || ''}
              onChange={(e) => onFiltersChange({
                ...filters,
                dateTo: e.target.value || null
              })}
              className="text-xs px-2 py-1 border rounded"
            />
          </div>
        </div>
        
        {/* Communication Types */}
        <div>
          <label className="block text-xs font-medium text-gray-700 mb-1">
            Communication Types
          </label>
          <div className="flex flex-wrap gap-1">
            {communicationTypes.map((type) => (
              <label key={type} className="flex items-center">
                <input
                  type="checkbox"
                  checked={filters.communicationTypes.includes(type)}
                  onChange={(e) => {
                    const newTypes = e.target.checked
                      ? [...filters.communicationTypes, type]
                      : filters.communicationTypes.filter(t => t !== type);
                    onFiltersChange({ ...filters, communicationTypes: newTypes });
                  }}
                  className="mr-1"
                />
                <span className="text-xs capitalize">{type}</span>
              </label>
            ))}
          </div>
        </div>
        
        {/* Sentiment Filter */}
        <div>
          <label className="block text-xs font-medium text-gray-700 mb-1">
            Sentiment
          </label>
          <select
            value={filters.sentimentFilter || ''}
            onChange={(e) => onFiltersChange({
              ...filters,
              sentimentFilter: e.target.value || null
            })}
            className="text-xs px-2 py-1 border rounded w-full"
          >
            <option value="">All</option>
            {sentimentOptions.map((sentiment) => (
              <option key={sentiment} value={sentiment}>
                {sentiment.charAt(0).toUpperCase() + sentiment.slice(1)}
              </option>
            ))}
          </select>
        </div>
      </div>
    </div>
  );
};
```

#### 5. Compliance Flags Management Component
```typescript
// components/communications/ComplianceFlagsManagement.tsx
interface ComplianceFlagsManagementProps {
  communicationId: string;
  flags: ComplianceFlag[];
  onFlagResolve?: (flagId: string, resolution: FlagResolution) => void;
  onFlagCreate?: (flag: NewComplianceFlag) => void;
}

const ComplianceFlagsManagement: React.FC<ComplianceFlagsManagementProps> = ({
  communicationId,
  flags,
  onFlagResolve,
  onFlagCreate
}) => {
  const [showCreateForm, setShowCreateForm] = useState(false);
  const [newFlag, setNewFlag] = useState<Partial<NewComplianceFlag>>({});

  const handleResolveFlag = async (flagId: string) => {
    const resolution = {
      resolution_notes: prompt('Enter resolution notes:') || '',
      resolved_by: 'current_user', // Get from auth context
      resolution_action: 'manual_resolution'
    };
    
    try {
      await fetch(`/api/resman/compliance-flags/${flagId}/resolve`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(resolution)
      });
      
      onFlagResolve?.(flagId, resolution);
    } catch (error) {
      console.error('Error resolving flag:', error);
    }
  };

  const handleCreateFlag = async () => {
    if (!newFlag.type || !newFlag.description || !newFlag.severity) return;
    
    try {
      await fetch(`/api/resman/communications/${communicationId}/compliance-flags`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newFlag)
      });
      
      onFlagCreate?.(newFlag as NewComplianceFlag);
      setNewFlag({});
      setShowCreateForm(false);
    } catch (error) {
      console.error('Error creating flag:', error);
    }
  };

  return (
    <div className="compliance-flags-management">
      <div className="header flex items-center justify-between">
        <h4 className="text-sm font-medium text-gray-900">Compliance Flags</h4>
        <button
          onClick={() => setShowCreateForm(!showCreateForm)}
          className="text-xs px-2 py-1 bg-red-100 text-red-800 rounded hover:bg-red-200"
        >
          Add Flag
        </button>
      </div>
      
      {showCreateForm && (
        <div className="create-flag-form mt-3 p-3 bg-gray-50 rounded">
          <div className="grid grid-cols-1 md:grid-cols-3 gap-2">
            <select
              value={newFlag.type || ''}
              onChange={(e) => setNewFlag({ ...newFlag, type: e.target.value })}
              className="text-xs px-2 py-1 border rounded"
            >
              <option value="">Select Type</option>
              <option value="gdpr_consent">GDPR Consent</option>
              <option value="opt_out">Opt Out</option>
              <option value="sensitive_data">Sensitive Data</option>
              <option value="regulatory">Regulatory</option>
            </select>
            
            <select
              value={newFlag.severity || ''}
              onChange={(e) => setNewFlag({ ...newFlag, severity: e.target.value as any })}
              className="text-xs px-2 py-1 border rounded"
            >
              <option value="">Select Severity</option>
              <option value="low">Low</option>
              <option value="medium">Medium</option>
              <option value="high">High</option>
              <option value="critical">Critical</option>
            </select>
            
            <button
              onClick={handleCreateFlag}
              className="text-xs px-2 py-1 bg-blue-600 text-white rounded hover:bg-blue-700"
            >
              Create
            </button>
          </div>
          
          <textarea
            value={newFlag.description || ''}
            onChange={(e) => setNewFlag({ ...newFlag, description: e.target.value })}
            placeholder="Flag description..."
            className="mt-2 w-full text-xs px-2 py-1 border rounded"
            rows={2}
          />
        </div>
      )}
      
      <div className="flags-list mt-3 space-y-2">
        {flags.map((flag) => (
          <div key={flag.id} className="flag-item p-2 border rounded">
            <div className="flex items-center justify-between">
              <div>
                <span className={`text-xs px-2 py-1 rounded ${
                  flag.severity === 'critical' ? 'bg-red-100 text-red-800' :
                  flag.severity === 'high' ? 'bg-orange-100 text-orange-800' :
                  flag.severity === 'medium' ? 'bg-yellow-100 text-yellow-800' :
                  'bg-gray-100 text-gray-800'
                }`}>
                  {flag.type.replace('_', ' ').toUpperCase()}
                </span>
                <span className="ml-2 text-xs text-gray-600">
                  {flag.severity}
                </span>
              </div>
              
              {!flag.is_resolved && (
                <button
                  onClick={() => handleResolveFlag(flag.id)}
                  className="text-xs px-2 py-1 bg-green-100 text-green-800 rounded hover:bg-green-200"
                >
                  Resolve
                </button>
              )}
            </div>
            
            <p className="text-xs text-gray-700 mt-1">{flag.description}</p>
            
            {flag.is_resolved && (
              <div className="text-xs text-gray-500 mt-1">
                Resolved by {flag.resolved_by} on {formatTimestamp(flag.resolved_at)}
              </div>
            )}
          </div>
        ))}
      </div>
    </div>
  );
};
```

#### 6. Real-time Communication Sync Component
```typescript
// components/communications/RealTimeCommunicationSync.tsx
interface RealTimeCommunicationSyncProps {
  prospectId: string;
  onNewCommunication?: (communication: Communication) => void;
  onCommunicationUpdate?: (communication: Communication) => void;
  onSyncConflict?: (conflict: SyncConflict) => void;
}

const RealTimeCommunicationSync: React.FC<RealTimeCommunicationSyncProps> = ({
  prospectId,
  onNewCommunication,
  onCommunicationUpdate,
  onSyncConflict
}) => {
  const [wsConnection, setWsConnection] = useState<WebSocket | null>(null);
  const [connectionStatus, setConnectionStatus] = useState<'connecting' | 'connected' | 'disconnected'>('disconnected');

  useEffect(() => {
    const connectWebSocket = () => {
      const ws = new WebSocket('/api/resman/communications/sync/websocket');
      
      ws.onopen = () => {
        setConnectionStatus('connected');
        // Subscribe to prospect-specific updates
        ws.send(JSON.stringify({
          type: 'subscribe',
          prospect_id: prospectId
        }));
      };
      
      ws.onmessage = (event) => {
        const message: RealTimeCommunicationSyncMessage = JSON.parse(event.data);
        
        switch (message.type) {
          case 'communication_log':
            if (message.sync_direction === 'from_resman') {
              onNewCommunication?.(message.data);
            } else {
              onCommunicationUpdate?.(message.data);
            }
            break;
            
          case 'sync_conflict':
            onSyncConflict?.(message.data);
            break;
        }
      };
      
      ws.onclose = () => {
        setConnectionStatus('disconnected');
        // Attempt to reconnect after 5 seconds
        setTimeout(connectWebSocket, 5000);
      };
      
      ws.onerror = (error) => {
        console.error('WebSocket error:', error);
        setConnectionStatus('disconnected');
      };
      
      setWsConnection(ws);
    };
    
    connectWebSocket();
    
    return () => {
      if (wsConnection) {
        wsConnection.close();
      }
    };
  }, [prospectId]);

  return (
    <div className="real-time-sync-indicator">
      <div className={`flex items-center space-x-2 ${
        connectionStatus === 'connected' ? 'text-green-600' :
        connectionStatus === 'connecting' ? 'text-yellow-600' :
        'text-red-600'
      }`}>
        <div className={`w-2 h-2 rounded-full ${
          connectionStatus === 'connected' ? 'bg-green-600' :
          connectionStatus === 'connecting' ? 'bg-yellow-600' :
          'bg-red-600'
        }`} />
        <span className="text-xs">
          {connectionStatus === 'connected' ? 'Real-time sync active' :
           connectionStatus === 'connecting' ? 'Connecting...' :
           'Sync disconnected'}
        </span>
      </div>
    </div>
  );
};
```

### State Management

#### 1. Communication Store (Zustand)
```typescript
// stores/communicationStore.ts
interface CommunicationState {
  communications: Communication[];
  searchResults: CommunicationSearchResult[];
  syncStatus: SyncStatus | null;
  complianceFlags: ComplianceFlag[];
  loading: boolean;
  error: string | null;
  
  // Actions
  fetchCommunications: (prospectId: string, filters?: CommunicationFilters) => Promise<void>;
  searchCommunications: (query: string, filters?: SearchFilters) => Promise<void>;
  logCommunication: (communication: LogCommunicationRequest) => Promise<void>;
  resolveComplianceFlag: (flagId: string, resolution: FlagResolution) => Promise<void>;
  createComplianceFlag: (flag: NewComplianceFlag) => Promise<void>;
  updateSyncStatus: (status: SyncStatus) => void;
  addCommunication: (communication: Communication) => void;
  updateCommunication: (communication: Communication) => void;
  clearError: () => void;
}

export const useCommunicationStore = create<CommunicationState>((set, get) => ({
  communications: [],
  searchResults: [],
  syncStatus: null,
  complianceFlags: [],
  loading: false,
  error: null,
  
  fetchCommunications: async (prospectId, filters) => {
    set({ loading: true, error: null });
    try {
      const response = await fetch(`/api/resman/communications/${prospectId}/history`, {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(filters)
      });
      
      const data = await response.json();
      set({
        communications: data.communications,
        syncStatus: data.sync_status,
        loading: false
      });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },
  
  searchCommunications: async (query, filters) => {
    set({ loading: true, error: null });
    try {
      const response = await fetch('/api/resman/communications/search', {
        method: 'GET',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ search_query: query, ...filters })
      });
      
      const data = await response.json();
      set({
        searchResults: data.results,
        loading: false
      });
    } catch (error) {
      set({ error: error.message, loading: false });
    }
  },
  
  logCommunication: async (communication) => {
    try {
      const response = await fetch('/api/resman/communications/log', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(communication)
      });
      
      const data = await response.json();
      if (data.success) {
        // Add to local state
        get().addCommunication(data.communication);
      }
    } catch (error) {
      set({ error: error.message });
    }
  },
  
  resolveComplianceFlag: async (flagId, resolution) => {
    try {
      await fetch(`/api/resman/compliance-flags/${flagId}/resolve`, {
        method: 'PUT',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(resolution)
      });
      
      // Update local state
      const { complianceFlags } = get();
      const updatedFlags = complianceFlags.map(flag =>
        flag.id === flagId ? { ...flag, is_resolved: true, ...resolution } : flag
      );
      set({ complianceFlags: updatedFlags });
    } catch (error) {
      set({ error: error.message });
    }
  },
  
  createComplianceFlag: async (flag) => {
    try {
      const response = await fetch(`/api/resman/communications/${flag.communication_id}/compliance-flags`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(flag)
      });
      
      const data = await response.json();
      // Add to local state
      const { complianceFlags } = get();
      set({ complianceFlags: [...complianceFlags, data.flag] });
    } catch (error) {
      set({ error: error.message });
    }
  },
  
  updateSyncStatus: (status) => set({ syncStatus: status }),
  addCommunication: (communication) => {
    const { communications } = get();
    set({ communications: [communication, ...communications] });
  },
  updateCommunication: (communication) => {
    const { communications } = get();
    const updated = communications.map(c =>
      c.id === communication.id ? communication : c
    );
    set({ communications: updated });
  },
  clearError: () => set({ error: null })
}));
```

### Integration with Chat System

#### 1. Enhanced Chat Modal with Communication Logging
```typescript
// components/chat/ChatModal.tsx (enhanced)
const ChatModal: React.FC<ChatModalProps> = ({ prospectId, conversationId }) => {
  const { addCommunication } = useCommunicationStore();
  
  const handleMessageSent = async (message: ChatMessage) => {
    // Existing chat logic...
    
    // Log communication to ResMan
    try {
      const logData = {
        prospect_id: prospectId,
        communication_data: {
          type: 'chat',
          sender_type: message.sender === 'user' ? 'prospect' : 'ai',
          content: message.content,
          content_summary: await generateMessageSummary(message.content),
          sentiment_score: await analyzeSentiment(message.content),
          key_topics: await extractKeyTopics(message.content),
          timestamp: message.timestamp,
          metadata: {
            message_id: message.id,
            conversation_id: conversationId,
            message_length: message.content.length,
            response_time_ms: message.response_time,
            ai_model_used: message.ai_model,
            confidence_score: message.confidence
          }
        },
        sync_direction: 'to_resman'
      };

      // Check for compliance flags
      const complianceFlags = await checkComplianceFlags(message.content);
      if (complianceFlags.length > 0) {
        logData.compliance_flags = complianceFlags;
      }

      // Log to ResMan
      await logCommunicationToResMan(logData);

      // Add to local state
      addCommunication({
        id: message.id,
        type: 'chat',
        sender_type: message.sender === 'user' ? 'prospect' : 'ai',
        content: message.content,
        timestamp: message.timestamp,
        sync_status: { status: 'success', direction: 'to_resman' }
      });

    } catch (error) {
      console.error('Error logging communication:', error);
      // Handle error (show notification, retry, etc.)
    }
  };

  return (
    <div className="chat-modal">
      {/* Existing chat UI */}
      <ChatInterface onMessageSent={handleMessageSent} />
      
      {/* Communication history sidebar */}
      <div className="communication-history-sidebar">
        <CommunicationHistory prospectId={prospectId} />
      </div>
    </div>
  );
};
```

### User Interaction Flows

#### 1. Communication Search and Filtering Flow
```typescript
// pages/communications/search.tsx
const CommunicationSearchPage: React.FC = () => {
  const { searchResults, searchCommunications, loading } = useCommunicationStore();
  const [searchQuery, setSearchQuery] = useState('');
  const [filters, setFilters] = useState<SearchFilters>({});

  const handleSearch = () => {
    if (searchQuery.trim()) {
      searchCommunications(searchQuery, filters);
    }
  };

  return (
    <div className="communication-search-page">
      <div className="search-header">
        <h1>Communication Search</h1>
        <CommunicationSearch
          onSearchResults={(results) => console.log('Search results:', results)}
          onSearchMetadata={(metadata) => console.log('Search metadata:', metadata)}
        />
      </div>
      
      <div className="search-results">
        {loading ? (
          <LoadingSpinner />
        ) : (
          <div className="results-grid">
            {searchResults.map((result) => (
              <CommunicationCard
                key={result.id}
                communication={result}
                showSyncStatus={true}
              />
            ))}
          </div>
        )}
      </div>
    </div>
  );
};
```

#### 2. Compliance Management Flow
```typescript
// pages/communications/compliance.tsx
const ComplianceManagementPage: React.FC = () => {
  const { complianceFlags, resolveComplianceFlag, createComplianceFlag } = useCommunicationStore();
  const [selectedCommunication, setSelectedCommunication] = useState<Communication | null>(null);

  return (
    <div className="compliance-management-page">
      <div className="page-header">
        <h1>Compliance Management</h1>
        <div className="compliance-summary">
          <ComplianceSummary flags={complianceFlags} />
        </div>
      </div>
      
      <div className="compliance-content">
        <div className="communications-list">
          <CommunicationHistory
            onCommunicationSelect={setSelectedCommunication}
          />
        </div>
        
        {selectedCommunication && (
          <div className="compliance-details">
            <ComplianceFlagsManagement
              communicationId={selectedCommunication.id}
              flags={selectedCommunication.compliance_flags || []}
              onFlagResolve={resolveComplianceFlag}
              onFlagCreate={createComplianceFlag}
            />
          </div>
        )}
      </div>
    </div>
  );
};
```

### Error Handling and User Feedback

#### 1. Error Boundary for Communication Components
```typescript
// components/communications/CommunicationErrorBoundary.tsx
class CommunicationErrorBoundary extends React.Component<
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
    console.error('Communication component error:', error, errorInfo);
    // Log to error tracking service
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="communication-error">
          <h3>Communication System Error</h3>
          <p>There was an error loading the communication data.</p>
          <button
            onClick={() => this.setState({ hasError: false, error: null })}
            className="retry-button"
          >
            Try Again
          </button>
        </div>
      );
    }

    return this.props.children;
  }
}
```

#### 2. Toast Notifications for Sync Status
```typescript
// components/communications/SyncStatusToast.tsx
const SyncStatusToast: React.FC<{ syncStatus: SyncStatus }> = ({ syncStatus }) => {
  const getToastConfig = (status: string) => {
    switch (status) {
      case 'success':
        return { type: 'success', message: 'Communication synced successfully' };
      case 'failed':
        return { type: 'error', message: 'Failed to sync communication' };
      case 'conflict':
        return { type: 'warning', message: 'Sync conflict detected' };
      default:
        return { type: 'info', message: 'Syncing communication...' };
    }
  };

  const config = getToastConfig(syncStatus.status);

  return (
    <Toast
      type={config.type}
      message={config.message}
      duration={3000}
    />
  );
};
```

### Performance Optimizations

#### 1. Virtualized Communication List
```typescript
// components/communications/VirtualizedCommunicationList.tsx
import { FixedSizeList as List } from 'react-window';

const VirtualizedCommunicationList: React.FC<{ communications: Communication[] }> = ({
  communications
}) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <CommunicationCard communication={communications[index]} />
    </div>
  );

  return (
    <List
      height={600}
      itemCount={communications.length}
      itemSize={120}
      width="100%"
    >
      {Row}
    </List>
  );
};
```

#### 2. Debounced Search
```typescript
// hooks/useDebouncedSearch.ts
const useDebouncedSearch = (callback: (query: string) => void, delay: number = 300) => {
  const [searchQuery, setSearchQuery] = useState('');
  
  useEffect(() => {
    const handler = setTimeout(() => {
      if (searchQuery.trim()) {
        callback(searchQuery);
      }
    }, delay);

    return () => {
      clearTimeout(handler);
    };
  }, [searchQuery, callback, delay]);

  return { searchQuery, setSearchQuery };
};
```

This frontend implementation provides a comprehensive UI for managing multi-channel communications with ResMan, including real-time sync, search capabilities, compliance management, and user-friendly error handling. 
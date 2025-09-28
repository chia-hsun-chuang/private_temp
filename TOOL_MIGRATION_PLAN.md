# Tool Migration Plan: Legacy to Instance/Gallery/Task Tracking System

## Overview
The migration transforms individual tool views from direct state management to a structured instance-based system with persistent storage, gallery integration, and centralized task tracking. This enables session persistence, instance management, and unified monitoring across all tools.

---

## A. Differences List (Migrated vs Legacy)

### **Data Models**
- **➕ Added**: `ToolInstancePayload` struct with Codable conformance for persistent state
- **➕ Added**: `ToolInstanceRef` for metadata (ID, timestamps, status, tags, displayName)
- **➕ Added**: `ToolInstanceManifest<P>` wrapper combining ref + payload
- **➕ Added**: `InstanceStatus` enum (.idle, .inProgress, .completed, .failed, .reviewed)
- **➕ Added**: Validation methods (`isValidForSubmission`, `validationErrors`)
- **🔄 Changed**: State variables moved from `@State` to payload properties
- **🔄 Changed**: API status tracking expanded (requestId, lastPollAt, provider, completedAt)

*Rationale: Enables persistence, validation, and structured instance lifecycle management*

### **Persistence**
- **➕ Added**: `ToolInstanceManager<P>` for CRUD operations with atomic writes
- **➕ Added**: JSON-based manifest storage with ISO8601 date encoding
- **➕ Added**: Instance index files for fast listing/filtering
- **➕ Added**: Directory structure: `Documents/Instances/{ToolName}/{InstanceID}/`
- **➕ Added**: Forward-compatible Codable implementation with graceful field handling
- **🔄 Changed**: File paths stored as relative paths from Image directory
- **❌ Removed**: Ephemeral state - everything persists across app restarts

*Rationale: Session continuity, crash recovery, and data integrity*

### **UI/State Management**
- **➕ Added**: `ToolInstanceViewModel` as `@StateObject` for reactive state management
- **➕ Added**: Instance lifecycle methods (onAppear, onDisappear, resumeActiveRequest)
- **➕ Added**: Smart polling continuation (keeps timers active during navigation)
- **➕ Added**: Debounced saving to reduce I/O overhead
- **➕ Added**: Instance actions (duplicate, delete, tag editing)
- **🔄 Changed**: Parameters bound to ViewModel methods instead of direct `@State`
- **🔄 Changed**: Loading state restoration from persisted payload
- **🔄 Changed**: Error handling with persistent error messages

*Rationale: Better separation of concerns, state restoration, and user experience*

### **Orchestration**
- **➕ Added**: TestAPITracker integration for centralized monitoring
- **➕ Added**: Request-to-instance mapping via filename patterns
- **➕ Added**: Background task synchronization with tracker state
- **➕ Added**: Asset registration for gallery integration
- **🔄 Changed**: API requests include instance ID in output filenames
- **🔄 Changed**: Status polling resumes automatically on app restart
- **🔄 Changed**: Timer management prevents premature cancellation during navigation

*Rationale: Unified monitoring, better task lifecycle management*

### **Tool Registration/Config**
- **➕ Added**: `ToolKind` enum for standardized tool identification
- **➕ Added**: Tool-specific managers with type safety
- **➕ Added**: Instance capacity and quota tracking capabilities
- **➕ Added**: Tool-specific color schemes and branding
- **🔄 Changed**: Tool identification via structured enum vs string literals

*Rationale: Type safety, consistent branding, resource management*

### **Logging/Telemetry**
- **➕ Added**: Comprehensive lifecycle logging (creation, loading, saving, deletion)
- **➕ Added**: API request/response debugging with request IDs
- **➕ Added**: File operation logging (saves, loads, path resolution)
- **➕ Added**: State transition logging (idle → inProgress → completed)
- **🔄 Enhanced**: Error logging with context and instance IDs

*Rationale: Better debugging, monitoring, and user support*

### **Permissions and Safety**
- **➕ Added**: Atomic file operations with temp files and replacement
- **➕ Added**: Directory existence validation before operations
- **➕ Added**: Graceful handling of missing/corrupted manifests
- **➕ Added**: Input validation before API submission
- **🔄 Enhanced**: Path normalization for iOS `/private` prefix handling

*Rationale: Data integrity, crash safety, cross-platform compatibility*

---

## B. Step-by-Step Migration Plan

### **Phase 1: Data Model Creation**
1. **Create Payload Schema**
   - Define `{Tool}InstancePayload` struct with all tool parameters
   - Add API integration fields (requestId, provider, apiStatus, etc.)
   - Add UI state fields (isLoading, errorMessage, etc.)
   - Implement Codable with forward-compatible decoder
   - Add validation methods and computed properties

2. **Define Tool Registration**
   - Add tool to `ToolKind` enum with display name and color
   - Create tool-specific manager instance
   - Register tool in main navigation/content view

### **Phase 2: Persistence Layer**
3. **Set Up Instance Manager**
   - Create `ToolInstanceManager<{Tool}InstancePayload>` instance
   - Configure JSON encoder/decoder with ISO8601 dates
   - Implement directory structure creation
   - Add CRUD operations with atomic writes

4. **Implement File Management**
   - Convert absolute paths to relative paths for portability
   - Implement smart image import (avoid duplicates in Image folder)
   - Add asset cleanup and storage size tracking
   - Handle iOS path normalization (`/private` prefix)

### **Phase 3: ViewModel Architecture**
5. **Create Instance ViewModel**
   - Implement `{Tool}InstanceViewModel` as ObservableObject
   - Add reactive parameter update methods with debounced saving
   - Implement API request lifecycle management
   - Add instance actions (duplicate, delete, tag management)

6. **Integrate State Management**
   - Replace `@State` variables with ViewModel properties
   - Implement state restoration from persisted payload
   - Add loading state management and error handling
   - Implement smart polling continuation during navigation

### **Phase 4: UI Migration**
7. **Convert View Architecture**
   - Create `{Tool}InstanceView` using ViewModel
   - Replace direct state bindings with ViewModel method calls
   - Add instance info display (ID, creation time, status)
   - Implement instance management actions (duplicate, delete)

8. **Update Parameter Controls**
   - Bind UI controls to ViewModel update methods
   - Add validation feedback and error display
   - Implement keyboard dismissal and accessibility
   - Add loading states and progress indicators

### **Phase 5: Integration**
9. **API Integration**
   - Update API requests to include instance ID in filenames
   - Integrate with TestAPITracker for centralized monitoring
   - Implement status polling with proper cleanup
   - Add asset registration for gallery integration

10. **Navigation Integration**
    - Add tool to instance creation flow
    - Integrate with History tab for instance management
    - Add gallery integration for output viewing
    - Implement deep linking to specific instances

### **Phase 6: Testing & Validation**
11. **Unit Testing**
    - Test payload validation and serialization
    - Test instance manager CRUD operations
    - Test ViewModel state management and API integration
    - Test file management and path handling

12. **Integration Testing**
    - Test full instance lifecycle (create → edit → generate → complete)
    - Test persistence across app restarts
    - Test navigation and state restoration
    - Test error handling and recovery scenarios

---

## C. File Worklist (Universal Names)

### **Core Data Models**
- **`ToolInstancePayload.swift`** - Codable payload schema with tool parameters and API state
- **`ToolInstanceRef.swift`** - Instance metadata (ID, timestamps, status, tags, displayName)  
- **`ToolInstanceManifest.swift`** - Wrapper combining reference and payload
- **`InstanceStatus.swift`** - Status enum with display names and color mapping
- **`ToolKind.swift`** - Tool identification enum with branding information

### **Persistence Layer**
- **`ToolInstanceManager.swift`** - Generic CRUD manager with atomic operations
- **`InstanceDirectoryHelpers.swift`** - File system utilities and path management
- **`InstanceError.swift`** - Error types for instance operations

### **ViewModel Architecture**
- **`ToolInstanceViewModel.swift`** - Reactive state management and API integration
- **`APIError.swift`** - API-specific error types and handling

### **View Layer**
- **`ToolInstanceView.swift`** - Main instance editing interface
- **`InstanceImagePicker.swift`** - Reusable image selection component
- **`InstanceParameterControls.swift`** - Tool-specific parameter UI components

### **Integration Components**
- **`InstanceHistoryView.swift`** - Cross-tool instance management interface
- **`InstanceStatisticsView.swift`** - Storage and usage analytics
- **`WorkflowAssetManager.swift`** - Gallery integration and asset registration

### **Testing Infrastructure**
- **`ToolInstancePayloadTests.swift`** - Payload validation and serialization tests
- **`ToolInstanceManagerTests.swift`** - CRUD operations and persistence tests
- **`ToolInstanceViewModelTests.swift`** - State management and API integration tests
- **`InstanceIntegrationTests.swift`** - End-to-end workflow testing

### **Configuration**
- **`InstanceConfiguration.swift`** - Tool-specific settings and capabilities
- **`InstanceDirectoryStructure.md`** - Documentation of file organization
- **`MigrationGuide.md`** - Step-by-step migration instructions

---

## D. Migration Checklist

### **Prerequisites**
- [ ] Identify all tool parameters and API integration points
- [ ] Document current state management and file handling patterns  
- [ ] Review existing error handling and validation logic
- [ ] Backup current tool implementation
- [ ] Set up development branch for migration work

### **Data Model Implementation**
- [ ] Create payload struct with all tool parameters
- [ ] Add API integration fields (requestId, provider, status, etc.)
- [ ] Add UI state fields (isLoading, errorMessage, etc.)
- [ ] Implement Codable with forward-compatible decoder
- [ ] Add validation methods and computed properties
- [ ] Add tool to ToolKind enum with display name and color
- [ ] Write unit tests for payload validation and serialization

### **Persistence Layer**
- [ ] Create ToolInstanceManager instance for the tool
- [ ] Configure JSON encoder/decoder with ISO8601 dates
- [ ] Implement directory structure creation and validation
- [ ] Add CRUD operations with atomic file writes
- [ ] Implement smart image import with duplicate detection
- [ ] Add storage size tracking and cleanup utilities
- [ ] Write unit tests for manager CRUD operations

### **ViewModel Architecture**
- [ ] Create ToolInstanceViewModel as ObservableObject
- [ ] Implement reactive parameter update methods
- [ ] Add debounced saving to reduce I/O overhead
- [ ] Implement API request lifecycle management
- [ ] Add state restoration from persisted payload
- [ ] Implement smart polling continuation during navigation
- [ ] Add instance actions (duplicate, delete, tag management)
- [ ] Write unit tests for state management and API integration

### **UI Migration**
- [ ] Create ToolInstanceView using ViewModel pattern
- [ ] Replace @State variables with ViewModel properties
- [ ] Bind UI controls to ViewModel update methods
- [ ] Add instance info display (ID, creation time, status)
- [ ] Implement instance management actions UI
- [ ] Add validation feedback and error display
- [ ] Implement loading states and progress indicators
- [ ] Add keyboard dismissal and accessibility features

### **Integration Hooks**
- [ ] Update API requests to include instance ID in filenames
- [ ] Integrate with TestAPITracker for centralized monitoring
- [ ] Implement asset registration for gallery integration
- [ ] Add tool to instance creation flow in main navigation
- [ ] Integrate with History tab for instance management
- [ ] Test deep linking to specific instances

### **Testing & Validation**
- [ ] Write integration tests for full instance lifecycle
- [ ] Test persistence across app restarts and crashes
- [ ] Test navigation and state restoration scenarios
- [ ] Test error handling and recovery mechanisms
- [ ] Validate file management and path handling
- [ ] Test API integration and status polling
- [ ] Perform user acceptance testing with real workflows

### **Documentation & Cleanup**
- [ ] Update tool documentation with instance capabilities
- [ ] Document any breaking changes or migration notes
- [ ] Remove or deprecate legacy tool view
- [ ] Update main navigation to use instance view
- [ ] Add tool to automated test suites
- [ ] Create user-facing documentation for new features

---

## E. Acceptance Criteria

### **Instance Lifecycle**
- [ ] **Create Instance**: User can create new instance with default parameters
- [ ] **Edit Parameters**: All tool parameters persist when modified and saved
- [ ] **Generate Content**: API requests execute successfully with proper tracking
- [ ] **View Results**: Generated content displays correctly in instance view
- [ ] **Restore Session**: Instance state restores correctly after app restart

### **Gallery Integration**
- [ ] **Asset Registration**: Generated content appears automatically in gallery
- [ ] **Metadata Preservation**: Tool, prompt, and generation parameters saved with assets
- [ ] **Search Functionality**: Assets findable by tool name, prompt, or metadata
- [ ] **Thumbnail Generation**: Visual previews available for all supported content types

### **Task Monitoring**
- [ ] **Status Updates**: Instance status updates automatically (idle → inProgress → completed)
- [ ] **Progress Tracking**: Real-time status visible in Task Monitor and History tabs
- [ ] **Error Handling**: Failed requests show clear error messages and allow retry
- [ ] **Background Processing**: Tasks continue when navigating away from tool view

### **Data Persistence**
- [ ] **Crash Recovery**: Instances and progress survive app crashes and force-quits
- [ ] **Storage Efficiency**: No duplicate files created when importing from Image folder
- [ ] **Data Integrity**: All instance data saves atomically without corruption
- [ ] **Migration Safety**: Existing data remains accessible after system updates

### **User Experience**
- [ ] **Instance Management**: Users can duplicate, delete, and tag instances
- [ ] **History Navigation**: Easy access to previous instances via History tab
- [ ] **Search & Filter**: Find instances by tool, status, tags, or creation date
- [ ] **Performance**: No UI blocking during file operations or API requests

---

## F. Risks & Mitigations

### **Data Migration Risks**
**Risk**: Existing user data loss during migration  
**Mitigation**: 
- Implement schema versioning with backward compatibility
- Create data backup before migration
- Use feature flags to enable gradual rollout
- Provide data export/import utilities

**Risk**: File system corruption during atomic operations  
**Mitigation**:
- Use temp files with atomic replacement
- Implement transaction-like operations with rollback
- Add file integrity validation
- Monitor disk space before operations

### **Performance Risks**
**Risk**: UI blocking during large file operations  
**Mitigation**:
- Use background queues for file I/O
- Implement debounced saving to reduce writes
- Add progress indicators for long operations
- Optimize JSON encoding/decoding

**Risk**: Memory usage from loading many instances  
**Mitigation**:
- Implement lazy loading of instance data
- Use pagination in History tab
- Cache only essential metadata
- Add memory pressure monitoring

### **Integration Risks**
**Risk**: TestAPITracker synchronization issues  
**Mitigation**:
- Implement robust request-to-instance mapping
- Add conflict resolution for duplicate requests
- Use consistent filename patterns
- Add debugging logs for tracking issues

**Risk**: Breaking changes to existing workflows  
**Mitigation**:
- Maintain backward compatibility where possible
- Use feature flags for gradual migration
- Provide clear migration documentation
- Add automated regression tests

### **Rollback Plan**
1. **Feature Flag Disable**: Immediately disable instance system via feature flag
2. **Legacy Fallback**: Restore original tool views from backup branch
3. **Data Preservation**: Keep instance data intact for future retry
4. **User Communication**: Notify users of temporary reversion and timeline
5. **Issue Resolution**: Fix identified problems before re-enabling

### **Dark Launch Strategy**
1. **Internal Testing**: Enable for development team only
2. **Beta Users**: Gradual rollout to volunteer beta testers
3. **Percentage Rollout**: Enable for 10% → 50% → 100% of users
4. **Monitoring**: Track crash rates, performance metrics, and user feedback
5. **Quick Rollback**: Ability to disable within minutes if issues detected

---

## Summary

This comprehensive migration plan provides a reusable blueprint for transforming any legacy tool in the VSXRCreate0824 project to the new Instance, Gallery, and Task Tracking system. The analysis of FluxKontextMax (migrated) vs FluxKontextPro (legacy) reveals the key architectural changes needed:

**Core Transformation**: From ephemeral state management to persistent instance-based architecture with structured data models, reactive ViewModels, and integrated monitoring.

**Key Benefits**: Session persistence, crash recovery, unified task monitoring, gallery integration, and enhanced user experience through instance management capabilities.

**Implementation Strategy**: Phased approach starting with data models, building persistence layer, migrating UI architecture, and finally integrating with existing systems.

**Risk Management**: Comprehensive rollback plan, feature flags for gradual deployment, and extensive testing to ensure data integrity and user experience quality.

This plan can be applied to any of the 60+ tools in the project, providing a standardized approach to modernizing the tool architecture while maintaining backward compatibility and user workflow continuity.

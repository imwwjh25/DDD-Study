```mermaid
classDiagram
    %% Value Objects
    class CellVoltage {
        -Double voltage
        +CellVoltage(voltage: Double)
        +getVoltage() Double
        +isOverVoltage() boolean
        +isUnderVoltage() boolean
        +isNormal() boolean
    }

    class CellTemperature {
        -Double temperature
        +CellTemperature(temperature: Double)
        +getTemperature() Double
        +isOverTemperature() boolean
        +isUnderTemperature() boolean
        +isNormal() boolean
    }

    class SOC {
        -Double value
        +SOC(value: Double)
        +getValue() Double
        +isLow() boolean
        +isHigh() boolean
        +isNormal() boolean
    }

    class SOH {
        -Double value
        +SOH(value: Double)
        +getValue() Double
        +isLow() boolean
        +isNormal() boolean
    }

    %% Entities
    class BatteryCell {
        -Long id
        -String cellId
        -Long moduleId
        -Double voltage
        -Double temperature
        -Double current
        -Double soc
        -Double soh
        -Integer cycleCount
        -Boolean healthy
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        +BatteryCell()
        +getCellVoltage() CellVoltage
        +getCellTemperature() CellTemperature
        +getSocObj() SOC
        +getSohObj() SOH
        +updateStatus(voltage: CellVoltage, temperature: CellTemperature, current: Double, soc: SOC)
        +incrementCycleCount()
        +checkHealth() boolean
    }

    class BatteryModule {
        -Long id
        -String moduleId
        -Long packId
        -Integer cellCount
        -Double moduleCapacity
        -Double moduleVoltage
        -Double totalSoc
        -Boolean allCellsHealthy
        -List~String~ faultyCellIds
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        +BatteryModule()
        +addCell(cellId: String)
        +removeCell(cellId: String)
        +updateHealthStatus()
        +calculateTotalSoc() Double
    }

    class BatteryPack {
        -Long id
        -String packId
        -String name
        -Double totalCapacity
        -Double nominalVoltage
        -Double maxChargeCurrent
        -Double maxDischargeCurrent
        -Double totalSoc
        -PackStatus status
        -Boolean healthy
        -List~String~ faultyModuleIds
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        +BatteryPack()
        +addModule(moduleId: String)
        +removeModule(moduleId: String)
        +updateHealthStatus()
        +calculateTotalSoc() Double
        +getPackData() PackData
    }

    class PackStatus {
        <<enumeration>>
        NORMAL
        CHARGING
        DISCHARGING
        STANDBY
        ERROR
    }

    class PackData {
        -Double totalSoc
        -Boolean healthy
        -List~String~ faultyModuleIds
        +PackData()
    }

    class AlarmEvent {
        -Long id
        -String alarmId
        -AlarmType alarmType
        -AlarmLevel alarmLevel
        -String message
        -String relatedEntityId
        -LocalDateTime triggerTime
        -LocalDateTime ackTime
        -LocalDateTime clearTime
        -AlarmStatus status
        -String ackUser
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        +AlarmEvent()
        +acknowledge(user: String)
        +clear()
        +isActive() boolean
    }

    class AlarmType {
        <<enumeration>>
        OVER_VOLTAGE
        UNDER_VOLTAGE
        OVER_TEMPERATURE
        UNDER_TEMPERATURE
        LOW_SOC
        HIGH_SOC
        LOW_SOH
        OVER_CURRENT
        COMMUNICATION_ERROR
        SYSTEM_ERROR
    }

    class AlarmLevel {
        <<enumeration>>
        INFO
        WARNING
        CRITICAL
        EMERGENCY
    }

    class AlarmStatus {
        <<enumeration>>
        ACTIVE
        ACKNOWLEDGED
        CLEARED
    }

    class SystemLog {
        -Long id
        -LogLevel logLevel
        -String message
        -String source
        -LocalDateTime timestamp
        -String userId
        -String requestId
        -LocalDateTime createdAt
        +SystemLog()
    }

    class LogLevel {
        <<enumeration>>
        DEBUG
        INFO
        WARNING
        ERROR
        FATAL
    }

    %% Domain Services
    class AlarmManagerService {
        -AlarmEventRepository alarmEventRepository
        +AlarmManagerService(alarmEventRepository: AlarmEventRepository)
        +checkAlarms(cell: BatteryCell) List~AlarmEvent~
        +triggerAlarm(alarm: AlarmEvent) AlarmEvent
        +clearAlarm(alarmId: String)
        +getCurrentAlarms() List~AlarmEvent~
    }

    class SocCalculatorService {
        +calculateSoc(voltage: Double, current: Double, temperature: Double) SOC
        +updateSoc(currentSoc: SOC, current: Double, timeElapsed: Double, capacity: Double) SOC
    }

    class SystemLogService {
        -SystemLogRepository systemLogRepository
        +SystemLogService(systemLogRepository: SystemLogRepository)
        +debug(message: String, source: String)
        +info(message: String, source: String)
        +warning(message: String, source: String)
        +error(message: String, source: String)
        +fatal(message: String, source: String)
    }

    %% Repository Interfaces
    class BatteryCellRepository {
        <<interface>>
        +save(cell: BatteryCell) BatteryCell
        +findById(id: Long) Optional~BatteryCell~
        +findByCellId(cellId: String) Optional~BatteryCell~
        +findByModuleId(moduleId: Long) List~BatteryCell~
        +findAll() List~BatteryCell~
        +delete(cell: BatteryCell)
    }

    class BatteryModuleRepository {
        <<interface>>
        +save(module: BatteryModule) BatteryModule
        +findById(id: Long) Optional~BatteryModule~
        +findByModuleId(moduleId: String) Optional~BatteryModule~
        +findByPackId(packId: Long) List~BatteryModule~
        +findAll() List~BatteryModule~
        +delete(module: BatteryModule)
    }

    class BatteryPackRepository {
        <<interface>>
        +save(pack: BatteryPack) BatteryPack
        +findById(id: Long) Optional~BatteryPack~
        +findByPackId(packId: String) Optional~BatteryPack~
        +findAll() List~BatteryPack~
        +delete(pack: BatteryPack)
    }

    class AlarmEventRepository {
        <<interface>>
        +save(alarm: AlarmEvent) AlarmEvent
        +findById(id: Long) Optional~AlarmEvent~
        +findByAlarmId(alarmId: String) Optional~AlarmEvent~
        +findByStatus(status: AlarmStatus) List~AlarmEvent~
        +findByTimeRange(startTime: LocalDateTime, endTime: LocalDateTime) List~AlarmEvent~
        +findAll() List~AlarmEvent~
        +delete(alarm: AlarmEvent)
    }

    class SystemLogRepository {
        <<interface>>
        +save(log: SystemLog) SystemLog
        +findById(id: Long) Optional~SystemLog~
        +findByLogLevel(logLevel: LogLevel) List~SystemLog~
        +findByTimeRange(startTime: LocalDateTime, endTime: LocalDateTime) List~SystemLog~
        +findAll() List~SystemLog~
        +delete(log: SystemLog)
    }

    %% DTOs
    class BatteryCellDTO {
        -Long id
        -String cellId
        -Long moduleId
        -Double voltage
        -Double temperature
        -Double current
        -Double soc
        -Double soh
        -Integer cycleCount
        -Boolean healthy
        +BatteryCellDTO()
    }

    class BatteryModuleDTO {
        -Long id
        -String moduleId
        -Long packId
        -Integer cellCount
        -Double moduleCapacity
        -Double moduleVoltage
        -Double totalSoc
        -Boolean allCellsHealthy
        -List~String~ faultyCellIds
        +BatteryModuleDTO()
    }

    class BatteryPackDTO {
        -Long id
        -String packId
        -String name
        -Double totalCapacity
        -Double nominalVoltage
        -Double maxChargeCurrent
        -Double maxDischargeCurrent
        -Double totalSoc
        -PackStatus status
        -Boolean healthy
        -List~String~ faultyModuleIds
        +BatteryPackDTO()
    }

    class AlarmEventDTO {
        -Long id
        -String alarmId
        -AlarmType alarmType
        -AlarmLevel alarmLevel
        -String message
        -String relatedEntityId
        -LocalDateTime triggerTime
        -LocalDateTime ackTime
        -LocalDateTime clearTime
        -AlarmStatus status
        -String ackUser
        +AlarmEventDTO()
    }

    %% Application Services
    class BatteryCellApplicationService {
        -BatteryCellRepository batteryCellRepository
        -BatteryModuleRepository batteryModuleRepository
        -BatteryMapper batteryMapper
        -AlarmManagerService alarmManagerService
        -SystemLogService systemLogService
        +createBatteryCell(moduleId: Long, dto: BatteryCellDTO) BatteryCellDTO
        +getBatteryCellById(id: Long) BatteryCellDTO
        +getBatteryCellByCellId(cellId: String) BatteryCellDTO
        +getBatteryCellsByModuleId(moduleId: Long) List~BatteryCellDTO~
        +getAllBatteryCells() List~BatteryCellDTO~
        +updateBatteryCellStatus(id: Long, dto: BatteryCellDTO) BatteryCellDTO
        +deleteBatteryCell(id: Long)
    }

    class BatteryModuleApplicationService {
        -BatteryModuleRepository batteryModuleRepository
        -BatteryPackRepository batteryPackRepository
        -BatteryMapper batteryMapper
        -SystemLogService systemLogService
        +createBatteryModule(packId: Long, dto: BatteryModuleDTO) BatteryModuleDTO
        +getBatteryModuleById(id: Long) BatteryModuleDTO
        +getBatteryModuleByModuleId(moduleId: String) BatteryModuleDTO
        +getBatteryModulesByPackId(packId: Long) List~BatteryModuleDTO~
        +getAllBatteryModules() List~BatteryModuleDTO~
        +deleteBatteryModule(id: Long)
    }

    class BatteryPackApplicationService {
        -BatteryPackRepository batteryPackRepository
        -BatteryMapper batteryMapper
        -SystemLogService systemLogService
        +createBatteryPack(dto: BatteryPackDTO) BatteryPackDTO
        +getBatteryPackById(id: Long) BatteryPackDTO
        +getBatteryPackByPackId(packId: String) BatteryPackDTO
        +getAllBatteryPacks() List~BatteryPackDTO~
        +updateBatteryPackStatus(id: Long, status: PackStatus) BatteryPackDTO
        +deleteBatteryPack(id: Long)
    }

    class AlarmEventApplicationService {
        -AlarmEventRepository alarmEventRepository
        -BatteryMapper batteryMapper
        +getAlarmEventById(id: Long) AlarmEventDTO
        +getAlarmEventByAlarmId(alarmId: String) AlarmEventDTO
        +getActiveAlarms() List~AlarmEventDTO~
        +getAlarmsByTimeRange(startTime: LocalDateTime, endTime: LocalDateTime) List~AlarmEventDTO~
        +acknowledgeAlarm(alarmId: String, user: String) AlarmEventDTO
        +clearAlarm(alarmId: String) AlarmEventDTO
    }

    %% Mappers
    class BatteryMapper {
        <<interface>>
        +toEntity(dto: BatteryCellDTO) BatteryCell
        +toDTO(entity: BatteryCell) BatteryCellDTO
        +toEntity(dto: BatteryModuleDTO) BatteryModule
        +toDTO(entity: BatteryModule) BatteryModuleDTO
        +toEntity(dto: BatteryPackDTO) BatteryPack
        +toDTO(entity: BatteryPack) BatteryPackDTO
        +toEntity(dto: AlarmEventDTO) AlarmEvent
        +toDTO(entity: AlarmEvent) AlarmEventDTO
        +toCellDTOList(entities: List~BatteryCell~) List~BatteryCellDTO~
        +toModuleDTOList(entities: List~BatteryModule~) List~BatteryModuleDTO~
        +toPackDTOList(entities: List~BatteryPack~) List~BatteryPackDTO~
        +toAlarmDTOList(entities: List~AlarmEvent~) List~AlarmEventDTO~
    }

    %% Infrastructure - Persistence
    class BatteryCellRepositoryImpl {
        -BatteryCellMapper batteryCellMapper
        +BatteryCellRepositoryImpl(batteryCellMapper: BatteryCellMapper)
    }

    class BatteryModuleRepositoryImpl {
        -BatteryModuleMapper batteryModuleMapper
        +BatteryModuleRepositoryImpl(batteryModuleMapper: BatteryModuleMapper)
    }

    class BatteryPackRepositoryImpl {
        -BatteryPackMapper batteryPackMapper
        +BatteryPackRepositoryImpl(batteryPackMapper: BatteryPackMapper)
    }

    class AlarmEventRepositoryImpl {
        -AlarmEventMapper alarmEventMapper
        +AlarmEventRepositoryImpl(alarmEventMapper: AlarmEventMapper)
    }

    class SystemLogRepositoryImpl {
        -SystemLogMapper systemLogMapper
        +SystemLogRepositoryImpl(systemLogMapper: SystemLogMapper)
    }

    class BatteryCellMapper {
        <<interface>>
    }

    class BatteryModuleMapper {
        <<interface>>
    }

    class BatteryPackMapper {
        <<interface>>
    }

    class AlarmEventMapper {
        <<interface>>
    }

    class SystemLogMapper {
        <<interface>>
    }

    class MetaObjectHandlerImpl {
        +insertFill(metaObject: MetaObject)
        +updateFill(metaObject: MetaObject)
    }

    %% Interface Layer - Controllers
    class BatteryCellController {
        -BatteryCellApplicationService batteryCellApplicationService
        +BatteryCellController(batteryCellApplicationService: BatteryCellApplicationService)
        +create(moduleId: Long, dto: BatteryCellDTO) ResponseEntity~BatteryCellDTO~
        +getById(id: Long) ResponseEntity~BatteryCellDTO~
        +getByCellId(cellId: String) ResponseEntity~BatteryCellDTO~
        +getByModuleId(moduleId: Long) ResponseEntity~List~BatteryCellDTO~~
        +getAll() ResponseEntity~List~BatteryCellDTO~~
        +updateStatus(id: Long, dto: BatteryCellDTO) ResponseEntity~BatteryCellDTO~
        +delete(id: Long) ResponseEntity~Void~
    }

    class BatteryModuleController {
        -BatteryModuleApplicationService batteryModuleApplicationService
        +BatteryModuleController(batteryModuleApplicationService: BatteryModuleApplicationService)
        +create(packId: Long, dto: BatteryModuleDTO) ResponseEntity~BatteryModuleDTO~
        +getById(id: Long) ResponseEntity~BatteryModuleDTO~
        +getByModuleId(moduleId: String) ResponseEntity~BatteryModuleDTO~
        +getByPackId(packId: Long) ResponseEntity~List~BatteryModuleDTO~~
        +getAll() ResponseEntity~List~BatteryModuleDTO~~
        +delete(id: Long) ResponseEntity~Void~
    }

    class BatteryPackController {
        -BatteryPackApplicationService batteryPackApplicationService
        +BatteryPackController(batteryPackApplicationService: BatteryPackApplicationService)
        +create(dto: BatteryPackDTO) ResponseEntity~BatteryPackDTO~
        +getById(id: Long) ResponseEntity~BatteryPackDTO~
        +getByPackId(packId: String) ResponseEntity~BatteryPackDTO~
        +getAll() ResponseEntity~List~BatteryPackDTO~~
        +updateStatus(id: Long, status: PackStatus) ResponseEntity~BatteryPackDTO~
        +delete(id: Long) ResponseEntity~Void~
    }

    class AlarmEventController {
        -AlarmEventApplicationService alarmEventApplicationService
        +AlarmEventController(alarmEventApplicationService: AlarmEventApplicationService)
        +getById(id: Long) ResponseEntity~AlarmEventDTO~
        +getByAlarmId(alarmId: String) ResponseEntity~AlarmEventDTO~
        +getActive() ResponseEntity~List~AlarmEventDTO~~
        +getByTimeRange(startTime: LocalDateTime, endTime: LocalDateTime) ResponseEntity~List~AlarmEventDTO~~
        +acknowledge(alarmId: String, user: String) ResponseEntity~AlarmEventDTO~
        +clear(alarmId: String) ResponseEntity~AlarmEventDTO~
    }

    class GlobalExceptionHandler {
        +handleRuntimeException(ex: RuntimeException) ResponseEntity~ErrorResponse~
        +handleValidationException(ex: MethodArgumentNotValidException) ResponseEntity~ErrorResponse~
    }

    class ErrorResponse {
        -LocalDateTime timestamp
        -Integer status
        -String error
        -String message
        -String path
        +ErrorResponse()
    }

    %% Relationships
    BatteryCell --> CellVoltage
    BatteryCell --> CellTemperature
    BatteryCell --> SOC
    BatteryCell --> SOH

    BatteryModule *-- BatteryCell
    BatteryPack *-- BatteryModule
    BatteryPack --> PackData
    BatteryPack --> PackStatus
    AlarmEvent --> AlarmType
    AlarmEvent --> AlarmLevel
    AlarmEvent --> AlarmStatus
    SystemLog --> LogLevel

    BatteryCellRepositoryImpl ..|> BatteryCellRepository
    BatteryModuleRepositoryImpl ..|> BatteryModuleRepository
    BatteryPackRepositoryImpl ..|> BatteryPackRepository
    AlarmEventRepositoryImpl ..|> AlarmEventRepository
    SystemLogRepositoryImpl ..|> SystemLogRepository

    BatteryCellRepositoryImpl --> BatteryCellMapper
    BatteryModuleRepositoryImpl --> BatteryModuleMapper
    BatteryPackRepositoryImpl --> BatteryPackMapper
    AlarmEventRepositoryImpl --> AlarmEventMapper
    SystemLogRepositoryImpl --> SystemLogMapper

    BatteryCellApplicationService --> BatteryCellRepository
    BatteryCellApplicationService --> BatteryModuleRepository
    BatteryCellApplicationService --> BatteryMapper
    BatteryCellApplicationService --> AlarmManagerService
    BatteryCellApplicationService --> SystemLogService

    BatteryModuleApplicationService --> BatteryModuleRepository
    BatteryModuleApplicationService --> BatteryPackRepository
    BatteryModuleApplicationService --> BatteryMapper
    BatteryModuleApplicationService --> SystemLogService

    BatteryPackApplicationService --> BatteryPackRepository
    BatteryPackApplicationService --> BatteryMapper
    BatteryPackApplicationService --> SystemLogService

    AlarmEventApplicationService --> AlarmEventRepository
    AlarmEventApplicationService --> BatteryMapper

    AlarmManagerService --> AlarmEventRepository
    SystemLogService --> SystemLogRepository

    BatteryCellController --> BatteryCellApplicationService
    BatteryModuleController --> BatteryModuleApplicationService
    BatteryPackController --> BatteryPackApplicationService
    AlarmEventController --> AlarmEventApplicationService

    BatteryMapper --> BatteryCellDTO
    BatteryMapper --> BatteryCell
    BatteryMapper --> BatteryModuleDTO
    BatteryMapper --> BatteryModule
    BatteryMapper --> BatteryPackDTO
    BatteryMapper --> BatteryPack
    BatteryMapper --> AlarmEventDTO
    BatteryMapper --> AlarmEvent
```

# ecycle-tracker
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/**
 * @title RecycleTracker
 * @dev A smart contract to track recycling activities and reward users
 */
contract RecycleTracker {
    
    // Struct to represent a recycling record
    struct RecyclingRecord {
        uint256 id;
        address user;
        string materialType;
        uint256 quantity; // in grams
        uint256 timestamp;
        uint256 rewardTokens;
        bool verified;
    }
    
    // State variables
    address public owner;
    uint256 private recordCounter;
    uint256 public totalRecycledWeight;
    uint256 public constant TOKENS_PER_GRAM = 1; // 1 token per gram recycled
    
    // Mappings
    mapping(uint256 => RecyclingRecord) public recyclingRecords;
    mapping(address => uint256) public userTokenBalances;
    mapping(address => uint256[]) public userRecordIds;
    mapping(string => bool) public supportedMaterials;
    
    // Events
    event RecyclingRecorded(
        uint256 indexed recordId,
        address indexed user,
        string materialType,
        uint256 quantity,
        uint256 rewardTokens
    );
    
    event RecordVerified(uint256 indexed recordId, address indexed verifier);
    event TokensRedeemed(address indexed user, uint256 amount);
    event MaterialAdded(string materialType);
    
    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can perform this action");
        _;
    }
    
    modifier validMaterial(string memory _materialType) {
        require(supportedMaterials[_materialType], "Material type not supported");
        _;
    }
    
    /**
     * @dev Constructor to initialize the contract
     */
    constructor() {
        owner = msg.sender;
        recordCounter = 0;
        totalRecycledWeight = 0;
        
        // Initialize supported materials
        supportedMaterials["plastic"] = true;
        supportedMaterials["paper"] = true;
        supportedMaterials["glass"] = true;
        supportedMaterials["metal"] = true;
        supportedMaterials["electronic"] = true;
    }
    
    /**
     * @dev Core Function 1: Record recycling activity
     * @param _materialType Type of material being recycled
     * @param _quantity Quantity in grams
     */
    function recordRecycling(
        string memory _materialType,
        uint256 _quantity
    ) public validMaterial(_materialType) {
        require(_quantity > 0, "Quantity must be greater than zero");
        
        recordCounter++;
        uint256 rewardTokens = _quantity * TOKENS_PER_GRAM;
        
        // Create recycling record
        recyclingRecords[recordCounter] = RecyclingRecord({
            id: recordCounter,
            user: msg.sender,
            materialType: _materialType,
            quantity: _quantity,
            timestamp: block.timestamp,
            rewardTokens: rewardTokens,
            verified: false
        });
        
        // Update user records
        userRecordIds[msg.sender].push(recordCounter);
        
        // Award tokens (pending verification)
        userTokenBalances[msg.sender] += rewardTokens;
        totalRecycledWeight += _quantity;
        
        emit RecyclingRecorded(recordCounter, msg.sender, _materialType, _quantity, rewardTokens);
    }
    
    /**
     * @dev Core Function 2: Verify recycling record (by authorized personnel)
     * @param _recordId ID of the record to verify
     */
    function verifyRecord(uint256 _recordId) public onlyOwner {
        require(_recordId > 0 && _recordId <= recordCounter, "Invalid record ID");
        require(!recyclingRecords[_recordId].verified, "Record already verified");
        
        recyclingRecords[_recordId].verified = true;
        
        emit RecordVerified(_recordId, msg.sender);
    }
    
    /**
     * @dev Core Function 3: Redeem tokens for rewards
     * @param _amount Amount of tokens to redeem
     */
    function redeemTokens(uint256 _amount) public {
        require(_amount > 0, "Amount must be greater than zero");
        require(userTokenBalances[msg.sender] >= _amount, "Insufficient token balance");
        
        // Deduct tokens from user balance
        userTokenBalances[msg.sender] -= _amount;
        
        // In a real implementation, this would trigger actual reward distribution
        // For now, we just emit an event
        emit TokensRedeemed(msg.sender, _amount);
    }
    
    /**
     * @dev Get user's recycling statistics
     * @param _user Address of the user
     * @return totalRecords Total number of records
     * @return totalWeight Total weight recycled
     * @return tokenBalance Current token balance
     */
    function getUserStats(address _user) 
        public 
        view 
        returns (
            uint256 totalRecords,
            uint256 totalWeight,
            uint256 tokenBalance
        ) 
    {
        totalRecords = userRecordIds[_user].length;
        tokenBalance = userTokenBalances[_user];
        
        // Calculate total weight for user
        for (uint256 i = 0; i < userRecordIds[_user].length; i++) {
            uint256 recordId = userRecordIds[_user][i];
            totalWeight += recyclingRecords[recordId].quantity;
        }
    }
    
    /**
     * @dev Add new supported material type (only owner)
     * @param _materialType New material type to support
     */
    function addSupportedMaterial(string memory _materialType) public onlyOwner {
        supportedMaterials[_materialType] = true;
        emit MaterialAdded(_materialType);
    }
    
    /**
     * @dev Get all record IDs for a user
     * @param _user Address of the user
     * @return Array of record IDs
     */
    function getUserRecords(address _user) public view returns (uint256[] memory) {
        return userRecordIds[_user];
    }
    
    /**
     * @dev Get total number of records in the system
     * @return Total number of recycling records
     */
    function getTotalRecords() public view returns (uint256) {
        return recordCounter;
    }
}

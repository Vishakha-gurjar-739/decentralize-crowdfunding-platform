// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

contract Project {
    struct Campaign {
        address payable creator;
        string title;
        string description;
        uint256 goalAmount;
        uint256 raisedAmount;
        uint256 deadline;
        bool isActive;
        bool goalReached;
    }
    
    struct Contribution {
        address contributor;
        uint256 amount;
        uint256 timestamp;
    }
    
    mapping(uint256 => Campaign) public campaigns;
    mapping(uint256 => Contribution[]) public campaignContributions;
    mapping(uint256 => mapping(address => uint256)) public contributorAmounts;
    
    uint256 public campaignCounter;
    uint256 public platformFeePercentage = 2; // 2% platform fee
    address payable public platformOwner;
    
    event CampaignCreated(
        uint256 indexed campaignId,
        address indexed creator,
        string title,
        uint256 goalAmount,
        uint256 deadline
    );
    
    event ContributionMade(
        uint256 indexed campaignId,
        address indexed contributor,
        uint256 amount
    );
    
    event FundsWithdrawn(
        uint256 indexed campaignId,
        address indexed creator,
        uint256 amount
    );
    
    event RefundIssued(
        uint256 indexed campaignId,
        address indexed contributor,
        uint256 amount
    );
    
    modifier onlyActiveCampaign(uint256 _campaignId) {
        require(campaigns[_campaignId].isActive, "Campaign is not active");
        require(block.timestamp < campaigns[_campaignId].deadline, "Campaign has ended");
        _;
    }
    
    modifier onlyCampaignCreator(uint256 _campaignId) {
        require(msg.sender == campaigns[_campaignId].creator, "Only campaign creator can call this");
        _;
    }
    
    constructor() {
        platformOwner = payable(msg.sender);
    }
    
    /**
     * @dev Creates a new crowdfunding campaign
     * @param _title Campaign title
     * @param _description Campaign description
     * @param _goalAmount Target amount to raise (in wei)
     * @param _durationInDays Campaign duration in days
     */
    function createCampaign(
        string memory _title,
        string memory _description,
        uint256 _goalAmount,
        uint256 _durationInDays
    ) external {
        require(_goalAmount > 0, "Goal amount must be greater than 0");
        require(_durationInDays > 0, "Duration must be greater than 0");
        require(bytes(_title).length > 0, "Title cannot be empty");
        
        uint256 deadline = block.timestamp + (_durationInDays * 1 days);
        
        campaigns[campaignCounter] = Campaign({
            creator: payable(msg.sender),
            title: _title,
            description: _description,
            goalAmount: _goalAmount,
            raisedAmount: 0,
            deadline: deadline,
            isActive: true,
            goalReached: false
        });
        
        emit CampaignCreated(campaignCounter, msg.sender, _title, _goalAmount, deadline);
        campaignCounter++;
    }
    
    /**
     * @dev Allows users to contribute to a campaign
     * @param _campaignId ID of the campaign to contribute to
     */
    function contributeToCampaign(uint256 _campaignId) 
        external 
        payable 
        onlyActiveCampaign(_campaignId) 
    {
        require(msg.value > 0, "Contribution must be greater than 0");
        require(_campaignId < campaignCounter, "Campaign does not exist");
        
        Campaign storage campaign = campaigns[_campaignId];
        
        campaign.raisedAmount += msg.value;
        contributorAmounts[_campaignId][msg.sender] += msg.value;
        
        campaignContributions[_campaignId].push(Contribution({
            contributor: msg.sender,
            amount: msg.value,
            timestamp: block.timestamp
        }));
        
        // Check if goal is reached
        if (campaign.raisedAmount >= campaign.goalAmount && !campaign.goalReached) {
            campaign.goalReached = true;
        }
        
        emit ContributionMade(_campaignId, msg.sender, msg.value);
    }
    
    /**
     * @dev Allows campaign creator to withdraw funds if goal is reached
     * @param _campaignId ID of the campaign
     */
    function withdrawFunds(uint256 _campaignId) 
        external 
        onlyCampaignCreator(_campaignId) 
    {
        Campaign storage campaign = campaigns[_campaignId];
        
        require(campaign.goalReached || block.timestamp >= campaign.deadline, 
                "Cannot withdraw: goal not reached and campaign still active");
        require(campaign.raisedAmount > 0, "No funds to withdraw");
        require(campaign.isActive, "Campaign is not active");
        
        uint256 totalAmount = campaign.raisedAmount;
        uint256 platformFee = (totalAmount * platformFeePercentage) / 100;
        uint256 creatorAmount = totalAmount - platformFee;
        
        campaign.raisedAmount = 0;
        campaign.isActive = false;
        
        // Transfer funds
        platformOwner.transfer(platformFee);
        campaign.creator.transfer(creatorAmount);
        
        emit FundsWithdrawn(_campaignId, campaign.creator, creatorAmount);
    }
    
    /**
     * @dev Allows contributors to get refund if campaign fails
     * @param _campaignId ID of the campaign
     */
    function getRefund(uint256 _campaignId) external {
        Campaign storage campaign = campaigns[_campaignId];
        
        require(block.timestamp >= campaign.deadline, "Campaign is still active");
        require(!campaign.goalReached, "Campaign goal was reached");
        require(contributorAmounts[_campaignId][msg.sender] > 0, "No contribution found");
        
        uint256 refundAmount = contributorAmounts[_campaignId][msg.sender];
        contributorAmounts[_campaignId][msg.sender] = 0;
        
        payable(msg.sender).transfer(refundAmount);
        
        emit RefundIssued(_campaignId, msg.sender, refundAmount);
    }
    
    // View functions
    function getCampaign(uint256 _campaignId) 
        external 
        view 
        returns (
            address creator,
            string memory title,
            string memory description,
            uint256 goalAmount,
            uint256 raisedAmount,
            uint256 deadline,
            bool isActive,
            bool goalReached
        ) 
    {
        Campaign memory campaign = campaigns[_campaignId];
        return (
            campaign.creator,
            campaign.title,
            campaign.description,
            campaign.goalAmount,
            campaign.raisedAmount,
            campaign.deadline,
            campaign.isActive,
            campaign.goalReached
        );
    }
    
    function getCampaignContributions(uint256 _campaignId) 
        external 
        view 
        returns (Contribution[] memory) 
    {
        return campaignContributions[_campaignId];
    }
    
    function getContributorAmount(uint256 _campaignId, address _contributor) 
        external 
        view 
        returns (uint256) 
    {
        return contributorAmounts[_campaignId][_contributor];
    }
    
    function getTotalCampaigns() external view returns (uint256) {
        return campaignCounter;
    }
    
    // Admin functions
    function updatePlatformFee(uint256 _newFeePercentage) external {
        require(msg.sender == platformOwner, "Only platform owner can update fee");
        require(_newFeePercentage <= 10, "Fee cannot exceed 10%");
        platformFeePercentage = _newFeePercentage;
    }
}

# Decentralized-Scientific-Research-Funding.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/utils/Counters.sol";

/**
 * @title DecentralizedResearchFunding
 * @dev A decentralized platform for funding scientific research projects
 */
contract DecentralizedResearchFunding is ReentrancyGuard, Ownable {
    using Counters for Counters.Counter;
    
    Counters.Counter private _projectIds;
    Counters.Counter private _milestoneIds;
    
    // Structs
    struct ResearchProject {
        uint256 id;
        address researcher;
        string title;
        string description;
        string researchArea;
        uint256 fundingGoal;
        uint256 currentFunding;
        uint256 deadline;
        ProjectStatus status;
        uint256 createdAt;
        string[] milestones;
        mapping(address => uint256) contributions;
        address[] contributors;
    }
    
    struct Milestone {
        uint256 id;
        uint256 projectId;
        string description;
        uint256 fundingAmount;
        bool completed;
        bool verified;
        uint256 completedAt;
        string evidenceHash; // IPFS hash for research data/evidence
    }
    
    struct Researcher {
        address researcherAddress;
        string name;
        string institution;
        string[] expertise;
        uint256 reputation;
        bool verified;
        uint256[] projectIds;
    }
    
    enum ProjectStatus {
        Active,
        Funded,
        InProgress,
        Completed,
        Cancelled
    }
    
    // State variables
    mapping(uint256 => ResearchProject) public projects;
    mapping(uint256 => Milestone) public milestones;
    mapping(address => Researcher) public researchers;
    mapping(address => bool) public verifiers; // Trusted verifiers for milestone completion
    
    uint256 public platformFeePercentage = 250; // 2.5%
    uint256 public constant MAX_FEE_PERCENTAGE = 1000; // 10%
    address public feeRecipient;
    
    // Events
    event ProjectCreated(uint256 indexed projectId, address indexed researcher, string title, uint256 fundingGoal);
    event ProjectFunded(uint256 indexed projectId, address indexed contributor, uint256 amount);
    event MilestoneCreated(uint256 indexed milestoneId, uint256 indexed projectId, string description);
    event MilestoneCompleted(uint256 indexed milestoneId, uint256 indexed projectId, string evidenceHash);
    event MilestoneVerified(uint256 indexed milestoneId, address indexed verifier);
    event FundsReleased(uint256 indexed projectId, uint256 amount);
    event ResearcherRegistered(address indexed researcher, string name, string institution);
    event ProjectStatusChanged(uint256 indexed projectId, ProjectStatus newStatus);
    
    // Modifiers
    modifier onlyResearcher(uint256 _projectId) {
        require(projects[_projectId].researcher == msg.sender, "Not the project researcher");
        _;
    }
    
    modifier onlyVerifier() {
        require(verifiers[msg.sender], "Not an authorized verifier");
        _;
    }
    
    modifier projectExists(uint256 _projectId) {
        require(_projectId > 0 && _projectId <= _projectIds.current(), "Project does not exist");
        _;
    }
    
    constructor(address _feeRecipient) Ownable(msg.sender) {
        feeRecipient = _feeRecipient;
        verifiers[msg.sender] = true; // Owner is initial verifier
    }
    
    /**
     * @dev Register as a researcher
     */
    function registerResearcher(
        string memory _name,
        string memory _institution,
        string[] memory _expertise
    ) external {
        require(bytes(_name).length > 0, "Name cannot be empty");
        require(bytes(_institution).length > 0, "Institution cannot be empty");
        
        Researcher storage researcher = researchers[msg.sender];
        researcher.researcherAddress = msg.sender;
        researcher.name = _name;
        researcher.institution = _institution;
        researcher.expertise = _expertise;
        researcher.reputation = 100; // Starting reputation
        
        emit ResearcherRegistered(msg.sender, _name, _institution);
    }
    
    /**
     * @dev Create a new research project
     */
    function createProject(
        string memory _title,
        string memory _description,
        string memory _researchArea,
        uint256 _fundingGoal,
        uint256 _durationInDays,
        string[] memory _milestones
    ) external returns (uint256) {
        require(bytes(_title).length > 0, "Title cannot be empty");
        require(_fundingGoal > 0, "Funding goal must be greater than 0");
        require(_durationInDays > 0, "Duration must be greater than 0");
        require(researchers[msg.sender].researcherAddress != address(0), "Must be registered researcher");
        
        _projectIds.increment();
        uint256 projectId = _projectIds.current();
        
        ResearchProject storage project = projects[projectId];
        project.id = projectId;
        project.researcher = msg.sender;
        project.title = _title;
        project.description = _description;
        project.researchArea = _researchArea;
        project.fundingGoal = _fundingGoal;
        project.deadline = block.timestamp + (_durationInDays * 1 days);
        project.status = ProjectStatus.Active;
        project.createdAt = block.timestamp;
        project.milestones = _milestones;
        
        researchers[msg.sender].projectIds.push(projectId);
        
        emit ProjectCreated(projectId, msg.sender, _title, _fundingGoal);
        return projectId;
    }
    
    /**
     * @dev Fund a research project
     */
    function fundProject(uint256 _projectId) external payable projectExists(_projectId) nonReentrant {
        require(msg.value > 0, "Funding amount must be greater than 0");
        
        ResearchProject storage project = projects[_projectId];
        require(project.status == ProjectStatus.Active, "Project is not active");
        require(block.timestamp < project.deadline, "Project deadline has passed");
        require(project.currentFunding < project.fundingGoal, "Project already fully funded");
        
        // Add to contributions
        if (project.contributions[msg.sender] == 0) {
            project.contributors.push(msg.sender);
        }
        project.contributions[msg.sender] += msg.value;
        project.currentFunding += msg.value;
        
        // Check if project is now fully funded
        if (project.currentFunding >= project.fundingGoal) {
            project.status = ProjectStatus.Funded;
            emit ProjectStatusChanged(_projectId, ProjectStatus.Funded);
        }
        
        emit ProjectFunded(_projectId, msg.sender, msg.value);
    }
    
    /**
     * @dev Create a milestone for a project
     */
    function createMilestone(
        uint256 _projectId,
        string memory _description,
        uint256 _fundingAmount
    ) external onlyResearcher(_projectId) projectExists(_projectId) returns (uint256) {
        require(bytes(_description).length > 0, "Description cannot be empty");
        require(_fundingAmount > 0, "Funding amount must be greater than 0");
        
        ResearchProject storage project = projects[_projectId];
        require(project.status == ProjectStatus.Funded || project.status == ProjectStatus.InProgress, 
                "Project must be funded or in progress");
        
        _milestoneIds.increment();
        uint256 milestoneId = _milestoneIds.current();
        
        Milestone storage milestone = milestones[milestoneId];
        milestone.id = milestoneId;
        milestone.projectId = _projectId;
        milestone.description = _description;
        milestone.fundingAmount = _fundingAmount;
        
        emit MilestoneCreated(milestoneId, _projectId, _description);
        return milestoneId;
    }
    
    /**
     * @dev Mark milestone as completed with evidence
     */
    function completeMilestone(
        uint256 _milestoneId,
        string memory _evidenceHash
    ) external {
        Milestone storage milestone = milestones[_milestoneId];
        require(milestone.id != 0, "Milestone does not exist");
        require(projects[milestone.projectId].researcher == msg.sender, "Not the project researcher");
        require(!milestone.completed, "Milestone already completed");
        require(bytes(_evidenceHash).length > 0, "Evidence hash cannot be empty");
        
        milestone.completed = true;
        milestone.completedAt = block.timestamp;
        milestone.evidenceHash = _evidenceHash;
        
        // Update project status to InProgress if first milestone
        ResearchProject storage project = projects[milestone.projectId];
        if (project.status == ProjectStatus.Funded) {
            project.status = ProjectStatus.InProgress;
            emit ProjectStatusChanged(milestone.projectId, ProjectStatus.InProgress);
        }
        
        emit MilestoneCompleted(_milestoneId, milestone.projectId, _evidenceHash);
    }
    
    /**
     * @dev Verify a completed milestone
     */
    function verifyMilestone(uint256 _milestoneId) external onlyVerifier {
        Milestone storage milestone = milestones[_milestoneId];
        require(milestone.id != 0, "Milestone does not exist");
        require(milestone.completed, "Milestone not completed");
        require(!milestone.verified, "Milestone already verified");
        
        milestone.verified = true;
        
        // Release funds for this milestone
        _releaseMilestoneFunds(_milestoneId);
        
        emit MilestoneVerified(_milestoneId, msg.sender);
    }
    
    /**
     * @dev Internal function to release milestone funds
     */
    function _releaseMilestoneFunds(uint256 _milestoneId) internal {
        Milestone storage milestone = milestones[_milestoneId];
        ResearchProject storage project = projects[milestone.projectId];
        
        uint256 amount = milestone.fundingAmount;
        uint256 fee = (amount * platformFeePercentage) / 10000;
        uint256 researcherAmount = amount - fee;
        
        // Transfer funds
        payable(project.researcher).transfer(researcherAmount);
        payable(feeRecipient).transfer(fee);
        
        // Update researcher reputation
        researchers[project.researcher].reputation += 10;
        
        emit FundsReleased(milestone.projectId, researcherAmount);
    }
    
    /**
     * @dev Get project details
     */
    function getProject(uint256 _projectId) external view projectExists(_projectId) returns (
        uint256 id,
        address researcher,
        string memory title,
        string memory description,
        string memory researchArea,
        uint256 fundingGoal,
        uint256 currentFunding,
        uint256 deadline,
        ProjectStatus status,
        uint256 createdAt
    ) {
        ResearchProject storage project = projects[_projectId];
        return (
            project.id,
            project.researcher,
            project.title,
            project.description,
            project.researchArea,
            project.fundingGoal,
            project.currentFunding,
            project.deadline,
            project.status,
            project.createdAt
        );
    }
    
    /**
     * @dev Get project contributors
     */
    function getProjectContributors(uint256 _projectId) external view projectExists(_projectId) returns (address[] memory) {
        return projects[_projectId].contributors;
    }
    
    /**
     * @dev Get contributor's contribution amount
     */
    function getContribution(uint256 _projectId, address _contributor) external view projectExists(_projectId) returns (uint256) {
        return projects[_projectId].contributions[_contributor];
    }
    
    /**
     * @dev Get researcher details
     */
    function getResearcher(address _researcher) external view returns (
        string memory name,
        string memory institution,
        string[] memory expertise,
        uint256 reputation,
        bool verified,
        uint256[] memory projectIds
    ) {
        Researcher storage researcher = researchers[_researcher];
        return (
            researcher.name,
            researcher.institution,
            researcher.expertise,
            researcher.reputation,
            researcher.verified,
            researcher.projectIds
        );
    }
    
    /**
     * @dev Add/remove verifier (only owner)
     */
    function setVerifier(address _verifier, bool _status) external onlyOwner {
        verifiers[_verifier] = _status;
    }
    
    /**
     * @dev Update platform fee (only owner)
     */
    function setPlatformFee(uint256 _feePercentage) external onlyOwner {
        require(_feePercentage <= MAX_FEE_PERCENTAGE, "Fee too high");
        platformFeePercentage = _feePercentage;
    }
    
    /**
     * @dev Get total number of projects
     */
    function getTotalProjects() external view returns (uint256) {
        return _projectIds.current();
    }
    
    /**
     * @dev Get total number of milestones
     */
    function getTotalMilestones() external view returns (uint256) {
        return _milestoneIds.current();
    }
    
    /**
     * @dev Emergency withdraw (only owner)
     */
    function emergencyWithdraw() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}

contact address: 0xEb867be86F06F7F16592f23cB37C2cCbEa1049d0

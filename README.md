
I am debugging a SailPoint IdentityIQ employee separation issue.

Flow completed successfully up to this point:
1. CHRIS termination transaction was added.
2. Single application account aggregation ran successfully.
3. Refresh Identity ran successfully.
4. Approved event was created on the identity.
5. Effective date was in the past.
6. Process Scheduled Events ran successfully.
7. Access Request was created.
8. Employee Separation Request Workflow succeeded.
9. LCM Target Provisioning succeeded.

Failure:
Task Result shows:
Separate User Workflow: ABuccitelli = FAIL

Error:
sailpoint.tools.GeneralException: The application script threw an exception:
java.lang.NullPointerException BSF info: script at line: 0 column: columnNo

Current Workflow Step:
Move AD Account

Workflow step:
<Step action="script:moveAccount(context, identityName, targetOuAttributeName,
'script:gov.fdic.sailpoint.constants.ActiveDirectoryConstants.TOMBSTONE_OU_SUFFIX',
identityRequestId)"
name="Move AD Account"
resultVariable="moveAccountPlan">

The method is in:
FDIC-RuleLibrary-MoveAccount.xml

Method:
public static void moveAccount(SailPointContext context, String identityIdOrName, String targetOuAttributeName, String tombstoneOuSuffix, String identityRequestId)

Please add safe null-checks and detailed log.error debug statements inside moveAccount() to identify which object is null before the NullPointerException occurs.

Add logs for:
- identityIdOrName
- targetOuAttributeName
- tombstoneOuSuffix
- identityRequestId
- identity lookup result
- application lookup result
- AD link result
- nativeIdentity/DN
- target OU attribute value
- identityRequest lookup result
- provisioning plan
- account request

Do not change business logic yet. Only add defensive logging and null checks that throw clear GeneralException messages instead of generic NullPointerException.

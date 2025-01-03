### Fuzz cases
1. total fund remains same even after switching role Participant -> Funder
2. 


===========================================================
### Manual findings
HIGH  
(high) `withdraw()` does not withdraw the native ether tokens. Hence, ether will be stuck in the contract forever after deadline has passed. - DONE

(high) `withdraw()` missing deadline check, allowing host to withdraw before deadline and users not able to get refund of ERC20 tokens - DONE

(high) depositing ether using `receive()` does not register the user as a participant - DONE

(high) reentrancy modifier not working as expected - DONE

(high) deposit 0 amount and be participant - DONE

(high) can become participant by `changeParticipationStatus()` calling before deadline

MED  
(med) if deadline is not set, users cannot deposit - DONE

(med) deadline can be set multiple times because deadlineSet is never set to True - DONE

(med) missing function to sign up for other users - DONE

LOW  
(low) transfer() in _refundETH() deprecated - DONE

(low) the first initial host cannot become the host again without depositing. - DONE







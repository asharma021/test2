That makes sense, Josh. I haven't made many changes yet since my main focus was getting the existing PostgreSQL Extract/Load pipeline running in DEV.

I'm good with creating an MR into development for the initial review. My only concern is that if we need to merge for every small change or test, it could slow down development. Would it make sense to eventually have a dedicated DEV testing runner or protected testing branch so we can validate changes independently while keeping the existing runner secure?

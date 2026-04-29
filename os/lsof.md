# find what's using port 8081
lsof -i :8081


# only show listening processes
lsof -i :8081 -sTCP:LISTEN


# get just the PID(s)
lsof -ti :8081
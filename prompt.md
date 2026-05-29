# prompt

i want to introduce a new logical layer. should be the last one.

"josyn-backend" 

it will be another strongly decoupled layer.

1. the new layer shall exist as a placeholder-stub in a new repo "josyn-backend"
2. the whole documentation shall be updated accordingly.

remember: the audience are collegue-architects which shall get a very good understanding of what currently exists in this poc and the future-vision.

## background information

### the old backend from a logical component-view:

#### JobSystem.Service 

a windows-service with a wcf-server which is listening for request to start a new job-session

#### JobSystem.TriggerAgent

a windows-service with polls ferquently (once a minute ) if a job execution is sheduled.

#### JobSystem.Sheduling

a properietary time-sheduling mechanism

#### JobSystem.WorkflowAdpater

a component which starts a job as an activity within an active workflow 
propiertary development too

#### JobSystem.Database

physically on an own SQL-server
has a table calls JobSessions. where every job-session lives and is documented. 

#### JobSystem.JobRepository

a file-system structure, where all the jobs are deployed

#### JobSystem.SessionStarter

a single implementation, compiled into the two services, which actuall starts a new job-session.
this is the rendevous of the backend with the rest of the system.

it is perfect candidate to for decoupling and for a step-by-step migration and a parallel existence of the old and the new system.

- the sessionstarter creates a new session-entry in the sessionstore-table
- then it just make an (impersonated) Process.Start on the last component, the old JobHost.exe with on argument: the fresh session-uid.

#### JobSystem.JobHost

this is the weakest point in the old architecture:

- the jobhost.exe exists once 
- for every job-session a new isolated process is spawned. so far-so good.

#### the flaws

the whole decoupling is thwarted here:

in the old-system, a job is povided as a class-library-dll in a sub-folder ni the job-repository, sided with its other needed assemblies
the job-host loads the job into its process-space with heavy use of reflection.
a lot of explicit Assembly-resolve stunts ar necessary here (it works...)
all already-loaded assemblies must be compatible with the assemblies, coming with the job.
job-host knows the database-logic of the session-store an has an own properietary dbaccess-layer.
it never can have ef-core, because it would collide with other ef-core-versions, the job would bring in.
the job-host must be provided an maintained in a separate version fpr every .Net-Version (.Net48, .Net8, .Net10...)

all glued-together. never to overcome limitations. a real evolution of the system halts here.

#### the cure

Sessionstarter will start a JapServer instead of the old JobHost directly.
Every Job *is* a JapClient.
Perfect decoupling of two worlds with this IPC-approach.
Backend can be refactored with its own pace.
Job-Development-Support can be refactores as a second evoltion path: mainly simpliying and modernising the whole Developer-View




































 







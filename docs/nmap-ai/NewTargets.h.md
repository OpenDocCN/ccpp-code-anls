# `nmap\NewTargets.h`

```cpp
/* $Id$ */

#ifndef NEWTARGETS_H
#define NEWTARGETS_H

#include <queue>
#include <set>
#include <string>


/* Adding new targets is for NSE scripts */
class NewTargets {
public:

  /* return a previous inserted target */
  static std::string read (void);

  /* get the number of all new added targets */
  static unsigned long get_number (void);

  /* get the number of queued targets left to scan */
  static unsigned long get_queued (void);

  /* Free the new_targets object. */
  static void free_new_targets (void);

  /* insert targets to the new_targets_queue */
  static unsigned long insert (const char *target);

private:
  NewTargets() {};

  /* A queue to push new targets that were discovered by NSE scripts.
   * Nmap will pop future targets from this queue. */
  std::queue<std::string> queue;

  /* A cache to save scanned targets specifications.
   * (These are targets that were pushed to Nmap scan queue) */
  std::set<std::string> history;

  /* Save new targets onto the queue */
  unsigned long push (const char *target);

  static NewTargets *new_targets;
};

#endif /* NEWTARGETS_H */
```
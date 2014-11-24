---
layout: post
category: "http"
title:  "Tomcat中session的安全问题"
tags: [tomcat,session]
---
本文是基于apache-tomcat-7.0.54。

  阅读了创建session的源码，发现有极小的可能性，session会被覆盖（不安全）。
  下面从ManagerBase的createSession方法开始
<pre class="prettyPrint">
@Override
public Session createSession(String sessionId) {
    if ((maxActiveSessions >= 0) &&
            (getActiveSessions() >= maxActiveSessions)) {
        rejectedSessions++;
        throw new TooManyActiveSessionsException(
                sm.getString("managerBase.createSession.ise"),
                maxActiveSessions);
    }
    // Recycle or create a Session instance
    Session session = createEmptySession();
    // Initialize the properties of the new session and return it
    session.setNew(true);
    session.setValid(true);
    session.setCreationTime(System.currentTimeMillis());
    session.setMaxInactiveInterval(this.maxInactiveInterval);
    String id = sessionId;
    if (id == null) {
        id = generateSessionId();
    }
    session.setId(id);
    sessionCounter++;
    SessionTiming timing = new SessionTiming(session.getCreationTime(), 0);
    synchronized (sessionCreationTiming) {
        sessionCreationTiming.add(timing);
        sessionCreationTiming.poll();
    }
    return (session);

}
</pre>
generateSessionId方法
<pre class="prettyPrint">
protected String generateSessionId() {
    String result = null;
    do {
        if (result != null) {
            // Not thread-safe but if one of multiple increments is lost
            // that is not a big deal since the fact that there was any
            // duplicate is a much bigger issue.
            duplicates++;
        }
        result = sessionIdGenerator.generateSessionId();
    } while (sessions.containsKey(result));
    
    return result;
}
</pre>
创建sessionId时会判断sessions（sessions是ConcurrentHashMap）中是否存在，若存在则一直循环，直到唯一。
新创建的session是session.setId(id)写入的。

假设这样一种情况，通过generateSessionId得到了第一个sessionIdA，这时还没有session.setId(sessionIdA)，
generateSessionId又创建了一个sessionIdB，恰巧sessionIdA=sessionIdB（这种几率非常低），这种情况下，前一个session就会被后一个session覆盖。

我们在SessionIdGenerator中写个main方法，测试下是否会出现sessionId重复的情况。
<pre class="prettyPrint">
public static void main(String[] args) {
	SessionIdGenerator generator = new SessionIdGenerator();
	generator.setSessionIdLength(5);
	Map<String, String> map = new HashMap<String, String>();
	for(int i=0; i< 100000000; i++) {
		String key = generator.generateSessionId();
		if(map.containsKey(key)) {
			System.out.println("i= " + i + "  " + key);
			break;
		}
		map.put(key, null);
	}
}
</pre>
sessionIdLength默认值为16，这种情况下，很难重复。于是我改为了5，在有限时间内重复一般都会出现。

结论：tomcat的session在极小概率的情况下会有问题。

若分析有误，希望提出来，谢谢。

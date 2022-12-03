# monotonic clock和wall clock

### 1.19版本之前暴露出问题

time api 原作者的一段话

> Go's current time API, which Rob Pike and I designed in 2011, defines an opaque type time.Time, a function time.Now that returns the current time, and a method t.Sub(u) to subtract two times, along with other methods interpreting a time.Time as a wall clock time. These are widely used by Go programs to measure elapsed times. The implementation of these functions only reads the system wall clock, never the monotonic clock, making the measurements incorrect in the event of clock resets.
>
> https://go.googlesource.com/proposal/+/master/design/12914-monotonic.md

翻译过来的意思是说，我们当时设计的时候用的是挂钟，不是单调时钟，结果大家都当做单调时钟来用，导致不准确。

```go
start := time.Now()
... something ...
end := time.Now()
elapsed := start.Sub(end)
```

> Go‘s original target was Google’s production servers, on which the wall clock never resets: the time is set very early in system startup, before any Go software runs, and leap seconds are handled by a leap smear, spreading the extra second over a 20-hour window in which the clock runs at 99.9986% speed (20 hours on that clock corresponds to 20 hours and one second in the real world). In 2011, I hoped that the trend toward reliable, reset-free computer clocks would continue and that Go programs could safely use the system wall clock to measure elapsed times. I was wrong. Although Akamai, Amazon, and Microsoft use leap smears now too, many systems still implement leap seconds by clock reset. A Go program measuring a negative elapsed time during a leap second caused CloudFlare's recent DNS outage. Wikipedia‘s list of examples of problems associated with the leap second now includes CloudFlare’s outage and notes Go's time APIs as the root cause. Beyond the problem of leap seconds, Go has also expanded to systems in non-production environments that may have less well-regulated clocks and consequently more frequent clock resets. Go must handle clock resets gracefully.

> https://go.googlesource.com/proposal/+/master/design/12914-monotonic.md

翻译过来是说，当初是给 google 服务器订制的，google的服务器是不会重置挂钟的，而其它系统都是通过重置挂钟来实现润秒，所以这个时间就不对了。

这会导致什么情况呢 time.Sleep(1\*time.Minute) 通常是睡1分钟，当挂钟被后置1小时，那么就会睡61分钟，有点以前夏令时感觉，这一天你会觉得多出来或者少了一小时。包括网络超时等都会有这个实际问题。

> 北京夏令时，是我国在1986年-1991年间实施的一种夏季时间制度，北京夏令时比标准的北京时间早一个小时。具体作法是：每年从四月中旬第一个星期日的凌晨2时整（北京时间），将时钟拨快一小时，即将表针由2时拨至3时，此时夏令时开始；到当年的九月中旬第一个星期日的凌晨2时整（北京夏令时），再将时钟拨回一小时，即将表针由2时拨至1时，此时夏令时结束。北京夏令时在我国从1986年开始实行，到1991年共六个年度，除1986年因是实行夏时制的第一年，从5月4日开始到9月14日结束外，其它年份均按规定的时段施行。在夏令时开始和结束前几天，新闻媒体均刊登有关部门的通告。1992年起，夏令时暂停实行。
>
> https://baike.baidu.com/item/%E5%8C%97%E4%BA%AC%E5%A4%8F%E4%BB%A4%E6%97%B6/1882131

### 1.19 版本改用单调时钟 Monotonic Clocks

> Starting from Go 1.19, the standard time package transparently uses Monotonic Clocks when necessary, so this package is no longer relevant.

This repository has been archived and is no longer maintained.

> https://github.com/gavv/monotime

翻译就是 自从1.19版本以后，就不需要维护了。

> Monotonic Clocks Operating systems provide both a “wall clock,” which is subject to changes for clock synchronization, and a “monotonic clock,” which is not. The general rule is that the wall clock is for telling time and the monotonic clock is for measuring time. Rather than split the API, in this package the Time returned by time.Now contains both a wall clock reading and a monotonic clock reading; later time-telling operations use the wall clock reading, but later time-measuring operations, specifically comparisons and subtractions, use the monotonic clock reading.
>
> https://pkg.go.dev/time@go1.19.2

翻译过来是说 现在的时间返回信息里包含了挂钟和单调时钟两种信息，报数用的是挂钟，但比较操作用的是单调时钟，尤其是sub。

```go
func TestTimeSub(t *testing.T) {
	start := time.Now()
	end := time.Now()
	elapsed := end.Sub(start)
	t.Log(elapsed)
}

func TestTimeSince(t *testing.T) {
	start := time.Now()
	t.Log(time.Since(start))
}

func TestTimeEcho(t *testing.T) {
	time := time.Now()
	t.Log(time.String())
	t.Log(time.UnixNano())
}
```

```
main_test.go:22: 2022-10-23 12:37:02.265758 +0800 CST m=+0.000640668
```

```go
func (t Time) String() string {
	s := t.Format("2006-01-02 15:04:05.999999999 -0700 MST")

	// Format monotonic clock reading as m=±ddd.nnnnnnnnn.
	if t.wall&hasMonotonic != 0 {
		m2 := uint64(t.ext)
		sign := byte('+')
		if t.ext < 0 {
			sign = '-'
			m2 = -m2
		}
		m1, m2 := m2/1e9, m2%1e9
		m0, m1 := m1/1e9, m1%1e9
		buf := make([]byte, 0, 24)
		buf = append(buf, " m="...)
		buf = append(buf, sign)
		wid := 0
		if m0 != 0 {
			buf = appendInt(buf, int(m0), 0)
			wid = 9
		}
		buf = appendInt(buf, int(m1), wid)
		buf = append(buf, '.')
		buf = appendInt(buf, int(m2), 9)
		s += string(buf)
	}
	return s
}
```

里面有个注释写的很清楚 如果返回有 m= 就是 monotonic clocks

```go
if t.wall&hasMonotonic != 0 {
}
```

### 到底什么是 monotonic clocks

```go
type Time struct {
	// wall and ext encode the wall time seconds, wall time nanoseconds,
	// and optional monotonic clock reading in nanoseconds.
	//
	// From high to low bit position, wall encodes a 1-bit flag (hasMonotonic),
	// a 33-bit seconds field, and a 30-bit wall time nanoseconds field.
	// The nanoseconds field is in the range [0, 999999999].
	// If the hasMonotonic bit is 0, then the 33-bit field must be zero
	// and the full signed 64-bit wall seconds since Jan 1 year 1 is stored in ext.
	// If the hasMonotonic bit is 1, then the 33-bit field holds a 33-bit
	// unsigned wall seconds since Jan 1 year 1885, and ext holds a
	// signed 64-bit monotonic clock reading, nanoseconds since process start.
	wall uint64
	ext  int64

	// loc specifies the Location that should be used to
	// determine the minute, hour, month, day, and year
	// that correspond to this Time.
	// The nil location means UTC.
	// All UTC times are represented with loc==nil, never loc==&utcLoc.
	loc *Location
}
```

例子

```go
func TestTimeEcho(t *testing.T) {
	t.Log(time.Now().String())
	time.Sleep(200 * time.Millisecond)
	t.Log(time.Now().String())
}
```

```
=== RUN   TestTimeEcho
    main_test.go:22: 2022-10-23 12:59:18.104868 +0800 CST m=+0.000500210
    main_test.go:25: 2022-10-23 12:59:18.30597 +0800 CST m=+0.201600668
--- PASS: TestTimeEcho (0.20s)
PASS

=== RUN   TestTimeEcho
    main_test.go:21: 2022-10-23 13:00:23.292602 +0800 CST m=+0.001739626
    main_test.go:23: 2022-10-23 13:00:23.493813 +0800 CST m=+0.202948960
--- PASS: TestTimeEcho (0.20s)
PASS
```

挂钟在向前走，单调时间永远是 m=+0.202948960，所以实际的执行时间是 两个单调时间的差值，是准确的。

### 参考

https://stackoverflow.com/questions/45791241/correctly-measure-time-duration-in-go

https://go.googlesource.com/proposal/+/master/design/12914-monotonic.md

https://pkg.go.dev/time#hdr-Monotonic\_Clocks

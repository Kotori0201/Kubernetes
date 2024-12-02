Kubernetes learning and training
#include<set>
#include<unordered_map>

using namespace std;

class Plane {
public:
	int start, end, length;
	Plane();

	Plane(int s, int e, int l)
	{
		start = s;
		end = e;
		length = l;
	}

	void setPlane(int s, int e, int l)
	{
		start = s;
		end = e;
		length = l;
	}
};

struct CMP {
	bool operator() (Plane a, Plane b) const
	{
		if (a.length == b.length)
		{
			return a.start < b.start;
		}
		return a.length > b.length;
	}
};

struct CMP2 {
	bool operator() (Plane a, Plane b) const
	{
		return a.start < b.start;
	}
};

set<Plane, CMP> freePlane;
set<Plane, CMP2> Building;
//set<Plane, CMP2> freePlaneCMP2;
int N;

void init(int n) {
	freePlane.clear();
	Building.clear();
	//freePlaneCMP2.clear();
	N = n;

	freePlane.insert({ 0, n - 1, n });
	//freePlaneCMP2.insert({ 0, n - 1, n });
}


int build(int mLength)
{
	set<Plane, CMP>::iterator it = freePlane.begin();

	for (it; it != freePlane.end(); it++)
	{
		if (it->length >= mLength)
		{
			break;
		}
	}

	if (it == freePlane.end())
	{
		return -1;
	}

	Plane tmp = *it;
	freePlane.erase(it);
	//freePlaneCMP2.erase(tmp);
	int remain_free = tmp.length - mLength;
	int half_remain_free = remain_free / 2;
	int s_new_building = tmp.start + half_remain_free;
	int e_new_building = tmp.start + half_remain_free + mLength - 1;
	Plane new_building({ s_new_building, e_new_building, mLength });

	Building.insert(new_building);

	if (remain_free > 0)
	{
		if (half_remain_free == 0)
		{
			Plane freeRight({ tmp.end, tmp.end, 1 });
			freePlane.insert(freeRight);
			//freePlaneCMP2.insert(freeRight);
		}
		else
		{
			Plane freeLeft({ tmp.start, tmp.start + half_remain_free - 1, half_remain_free });
			Plane freeRight({ e_new_building + 1, tmp.end, tmp.end - e_new_building});

			freePlane.insert(freeLeft);
			freePlane.insert(freeRight);
			//freePlaneCMP2.insert(freeRight);
			//freePlaneCMP2.insert(freeLeft);
		}
	}

	return s_new_building;
}

int demolish(int mAddr)
{
	Plane tmp(mAddr, 0, 0);
	int res = -1;
	auto a = Building.lower_bound(tmp);

	if (a == Building.end() || Building.empty())
	{
		return -1;
	}

	if (a->start == mAddr)
	{
		res = a->length;
		tmp = *a;
		Building.erase(a);
	}

	else if (a->start > mAddr)
	{
		a--;
		if (a != Building.begin() && a->start != NULL && a->start < mAddr && a->end >= mAddr)
		{
			res = a->length;
			tmp = *a;
			Building.erase(a);
		}
	}

	// Find left
	set<Plane, CMP>::iterator it = freePlane.begin();
	Plane l(0, 0, 0);
	Plane r(0, 0, 0);
	bool fl, fr;
	fl = fr = false;
	for (it; it != freePlane.end(); it++)
	{
		if (it->end == tmp.start - 1)
		{
			l = *it;
			fl = true;
		}

		if (it->start == tmp.end + 1)
		{
			r = *it;
			fr = true;
		}

		if (fl && fr)	break;
	}
	if (fl) {
		Plane resizefreePlane(0,0,0);
		resizefreePlane.start = l.start;
		resizefreePlane.end = tmp.end;
		resizefreePlane.length = l.length + tmp.length;
		tmp = resizefreePlane;
		freePlane.erase(l);
	}
	// Find right
	if (fr)
	{
		Plane resizefreePlane(0, 0, 0);
		resizefreePlane.start = tmp.start;
		resizefreePlane.end = r.end;
		resizefreePlane.length = r.length + tmp.length;
		tmp = resizefreePlane;
		freePlane.erase(r);
	}

	freePlane.insert(tmp);
	return res;
}
